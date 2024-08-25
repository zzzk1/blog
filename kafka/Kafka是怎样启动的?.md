# Kafka 启动追踪

## 从加载配置文件开始

### `Kafka.scala` 一切开始的开始

首先定位 main 函数最关键的三行代码: 

```scala
def main(args: Array[String]): Unit = {
  try {
    ...
    val serverProps = getProsFromArgs(args)
    val server = buildServer(serverProps)
    ...
    try server.startup()
    ...
  }
}
```

#### 接收 main 函数传递的参数构建启动服务所需参数

```scala
def getPropsFromArgs(args: Array[String]): Properties = {
  val optionParser = new OptionParser(false)
  val overrideOpt = optionParser.accepts("override", "Optional property that should override values set in server.properties file")
      .withRequiredArg()
      .ofType(classOf[String])
    optionParser.accepts("version", "Print version information and exit.")
  	// 这里的参数是我们输入的 shell 命令, 所以进行对应的检验
    if (args.isEmpty || args.contains("--help")) {
      CommandLineUtils.printUsageAndExit(optionParser,
        "USAGE: java [options] %s server.properties [--override 		 property=value]*".format(this.getClass.getCanonicalName.split('$').head))
    }

    if (args.contains("--version")) {
      CommandLineUtils.printVersionAndExit()
    }

    val props = Utils.loadProps(args(0))

    if (args.length > 1) {
      val options = optionParser.parse(args.slice(1, args.length): _*)

      if (options.nonOptionArguments().size() > 0) {
        CommandLineUtils.printUsageAndExit(optionParser, "Found non argument parameters: " + options.nonOptionArguments().toArray.mkString(","))
      }

      props ++= CommandLineUtils.parseKeyValueArgs(options.valuesOf(overrideOpt))
    }
    props
  }
```

#### 根据参数构建服务

```scala
// 这里会根据配置中的参数来决定使用 zookeeper 还是 Kraft 模式来保障一致性
private def buildServer(props: Properties): Server = {
  val config = KafkaConfig.fromProps(props, doLog = false)
  if (config.requiresZookeeper) {
    new KafkaServer(
      config,
      Time.SYSTEM,
      threadNamePrefix = None,
      enableForwarding = enableApiForwarding(config)
    )
  } else {
    new KafkaRaftServer(
      config,
      Time.SYSTEM,
    )
  }
}
```

####  Kraft 模式的启动

```scala
//kafkaRaftServer.scala
private val controller: Option[ControllerServer] = if (config.processRoles.contains(ProcessRole.ControllerRole)) {
  Some(new ControllerServer(
     sharedServer,
     KafkaRaftServer.configSchema,
     bootstrapMetadata,
   ))
} else {
   None
}

private val broker: Option[BrokerServer] = if (config.processRoles.contains(ProcessRole.BrokerRole)) {
    Some(new BrokerServer(sharedServer))
} else {
  None
}

override def startup(): Unit = {
    ...
    controller.foreach(_.startup())
    broker.foreach(_.startup())
    AppInfoParser.registerAppInfo(Server.MetricsPrefix, config.brokerId.toString, metrics, 			time.milliseconds())
    info(KafkaBroker.STARTED_MESSAGE)
  	...
  }

```

在这里, controller 要先于 broker 启动目的是保障 controller 的 endpoint 能被收集 (这里应该是 raft 模式下节点之间需要通信例如: 领导选举、心跳检测之类的 )

#### 构建 Controller (voters)

```scala
def startup(): Unit = {
    if (!maybeChangeStatus(SHUTDOWN, STARTING)) return
    val startupDeadline = Deadline.fromDelay(time, config.serverMaxStartupTimeMs, TimeUnit.MILLISECONDS)
    try {
      this.logIdent = logContext.logPrefix()
      info("Starting controller")
      config.dynamicConfig.initialize(zkClientOpt = None, clientMetricsReceiverPluginOpt = None)

      maybeChangeStatus(STARTING, STARTED)

      metricsGroup.newGauge("ClusterId", () => clusterId)
      metricsGroup.newGauge("yammer-metrics-count", () =>  KafkaYammerMetrics.defaultRegistry.allMetrics.size)

      linuxIoMetricsCollector = new LinuxIoMetricsCollector("/proc", time)
      if (linuxIoMetricsCollector.usable()) {
        metricsGroup.newGauge("linux-disk-read-bytes", () => linuxIoMetricsCollector.readBytes())
        metricsGroup.newGauge("linux-disk-write-bytes", () => linuxIoMetricsCollector.writeBytes())
      }

      authorizer = config.createNewAuthorizer()
      authorizer.foreach(_.configure(config.originals))

      metadataCache = MetadataCache.kRaftMetadataCache(config.nodeId, () => raftManager.client.kraftVersion())

      metadataCachePublisher = new KRaftMetadataCachePublisher(metadataCache)

      featuresPublisher = new FeaturesPublisher(logContext)

      registrationsPublisher = new ControllerRegistrationsPublisher()

      incarnationId = Uuid.randomUuid()

      val apiVersionManager = new SimpleApiVersionManager(
        ListenerType.CONTROLLER,
        config.unstableApiVersionsEnabled,
        config.migrationEnabled,
        () => featuresPublisher.features()
      )

      tokenCache = new DelegationTokenCache(ScramMechanism.mechanismNames)
      credentialProvider = new CredentialProvider(ScramMechanism.mechanismNames, tokenCache)
      socketServer = new SocketServer(config,
        metrics,
        time,
        credentialProvider,
        apiVersionManager)

      val listenerInfo = ListenerInfo
        .create(config.effectiveAdvertisedControllerListeners.map(_.toJava).asJava)
        .withWildcardHostnamesResolved()
        .withEphemeralPortsCorrected(name => socketServer.boundPort(new ListenerName(name)))
      socketServerFirstBoundPortFuture.complete(listenerInfo.firstListener().port())

      val endpointReadyFutures = {
        val builder = new EndpointReadyFutures.Builder()
        builder.build(authorizer.asJava,
          new KafkaAuthorizerServerInfo(
            new ClusterResource(clusterId),
            config.nodeId,
            listenerInfo.listeners().values(),
            listenerInfo.firstListener(),
            config.earlyStartListeners.map(_.value()).asJava))
      }

      sharedServer.startForController(listenerInfo)

      createTopicPolicy = Option(config.
        getConfiguredInstance(CREATE_TOPIC_POLICY_CLASS_NAME_CONFIG, classOf[CreateTopicPolicy]))
      alterConfigPolicy = Option(config.
        getConfiguredInstance(ALTER_CONFIG_POLICY_CLASS_NAME_CONFIG, classOf[AlterConfigPolicy]))

      val voterConnections = FutureUtils.waitWithLogging(logger.underlying, logIdent,
        "controller quorum voters future",
        sharedServer.controllerQuorumVotersFuture,
        startupDeadline, time)
      //这里就是当前集群中所有的 voters
      val controllerNodes = QuorumConfig.voterConnectionsToNodes(voterConnections)
      val quorumFeatures = new QuorumFeatures(config.nodeId,
        QuorumFeatures.defaultFeatureMap(config.unstableFeatureVersionsEnabled),
        controllerNodes.asScala.map(node => Integer.valueOf(node.id())).asJava)

      val delegationTokenKeyString = {
        if (config.tokenAuthEnabled) {
          config.delegationTokenSecretKey.value
        } else {
          null
        }
      }

      val controllerBuilder = {
        val leaderImbalanceCheckIntervalNs = if (config.autoLeaderRebalanceEnable) {
          OptionalLong.of(TimeUnit.NANOSECONDS.convert(config.leaderImbalanceCheckIntervalSeconds, TimeUnit.SECONDS))
        } else {
          OptionalLong.empty()
        }

        val maxIdleIntervalNs = config.metadataMaxIdleIntervalNs.fold(OptionalLong.empty)(OptionalLong.of)

        quorumControllerMetrics = new QuorumControllerMetrics(Optional.of(KafkaYammerMetrics.defaultRegistry), time, config.migrationEnabled)

        new QuorumController.Builder(config.nodeId, sharedServer.clusterId).
          setTime(time).
          setThreadNamePrefix(s"quorum-controller-${config.nodeId}-").
          setConfigSchema(configSchema).
          setRaftClient(raftManager.client).
          setQuorumFeatures(quorumFeatures).
          setDefaultReplicationFactor(config.defaultReplicationFactor.toShort).
          setDefaultNumPartitions(config.numPartitions.intValue()).
          setDefaultMinIsr(config.minInSyncReplicas.intValue()).
          setSessionTimeoutNs(TimeUnit.NANOSECONDS.convert(config.brokerSessionTimeoutMs.longValue(),
            TimeUnit.MILLISECONDS)).
          setLeaderImbalanceCheckIntervalNs(leaderImbalanceCheckIntervalNs).
          setMaxIdleIntervalNs(maxIdleIntervalNs).
          setMetrics(quorumControllerMetrics).
          setCreateTopicPolicy(createTopicPolicy.asJava).
          setAlterConfigPolicy(alterConfigPolicy.asJava).
          setConfigurationValidator(new ControllerConfigurationValidator(sharedServer.brokerConfig)).
          setStaticConfig(config.originals).
          setBootstrapMetadata(bootstrapMetadata).
          setFatalFaultHandler(sharedServer.fatalQuorumControllerFaultHandler).
          setNonFatalFaultHandler(sharedServer.nonFatalQuorumControllerFaultHandler).
          setZkMigrationEnabled(config.migrationEnabled).
          setDelegationTokenCache(tokenCache).
          setDelegationTokenSecretKey(delegationTokenKeyString).
          setDelegationTokenMaxLifeMs(config.delegationTokenMaxLifeMs).
          setDelegationTokenExpiryTimeMs(config.delegationTokenExpiryTimeMs).
          setDelegationTokenExpiryCheckIntervalMs(config.delegationTokenExpiryCheckIntervalMs).
          setEligibleLeaderReplicasEnabled(config.elrEnabled)
      }
      //创建能够参与选举的 controller, 成功的就是 leader (active controller) 失败的称为 follwer.
      controller = controllerBuilder.build()

      authorizer match {
        case Some(a: ClusterMetadataAuthorizer) => a.setAclMutator(controller)
        case _ =>
      }

      if (config.migrationEnabled) {
        val zkClient = KafkaZkClient.createZkClient("KRaft Migration", time, config, KafkaServer.zkClientConfigFromKafkaConfig(config))
        val zkConfigEncoder = config.passwordEncoderSecret match {
          case Some(secret) => PasswordEncoder.encrypting(secret,
            config.passwordEncoderKeyFactoryAlgorithm,
            config.passwordEncoderCipherAlgorithm,
            config.passwordEncoderKeyLength,
            config.passwordEncoderIterations)
          case None => PasswordEncoder.NOOP
        }
        val migrationClient = ZkMigrationClient(zkClient, zkConfigEncoder)
        val propagator: LegacyPropagator = new MigrationPropagator(config.nodeId, config)
        val migrationDriver = KRaftMigrationDriver.newBuilder()
          .setNodeId(config.nodeId)
          .setZkRecordConsumer(controller.asInstanceOf[QuorumController].zkRecordConsumer())
          .setZkMigrationClient(migrationClient)
          .setPropagator(propagator)
          .setInitialZkLoadHandler(publisher => sharedServer.loader.installPublishers(java.util.Collections.singletonList(publisher)))
          .setFaultHandler(sharedServer.faultHandlerFactory.build(
            "zk migration",
            fatal = false,
            () => {}
          ))
          .setQuorumFeatures(quorumFeatures)
          .setConfigSchema(configSchema)
          .setControllerMetrics(quorumControllerMetrics)
          .setMinMigrationBatchSize(config.migrationMetadataMinBatchSize)
          .setTime(time)
          .build()
        migrationDriver.start()
        migrationSupport = Some(ControllerMigrationSupport(zkClient, migrationDriver, propagator))
      }

      quotaManagers = QuotaFactory.instantiate(config,
        metrics,
        time,
        s"controller-${config.nodeId}-")
      clientQuotaMetadataManager = new ClientQuotaMetadataManager(quotaManagers, socketServer.connectionQuotas)
      //这里很重要, 用于决定将请求交给哪一个 handler 处理.
      controllerApis = new ControllerApis(socketServer.dataPlaneRequestChannel,
        authorizer,
        quotaManagers,
        time,
        controller,
        raftManager,
        config,
        clusterId,
        registrationsPublisher,
        apiVersionManager,
        metadataCache)
      //使用线程池, 接收客户端请求并处理
      controllerApisHandlerPool = new KafkaRequestHandlerPool(config.nodeId,
        socketServer.dataPlaneRequestChannel,
        controllerApis,
        time,
        config.numIoThreads,
        s"${DataPlaneAcceptor.MetricPrefix}RequestHandlerAvgIdlePercent",
        DataPlaneAcceptor.ThreadPrefix,
        "controller")

      // Set up the metadata cache publisher.
      metadataPublishers.add(metadataCachePublisher)

      // Set up the metadata features publisher.
      metadataPublishers.add(featuresPublisher)

      // Set up the controller registrations publisher.
      metadataPublishers.add(registrationsPublisher)

      // Create the registration manager, which handles sending KIP-919 controller registrations.
      registrationManager = new ControllerRegistrationManager(config.nodeId,
        clusterId,
        time,
        s"controller-${config.nodeId}-",
        QuorumFeatures.defaultFeatureMap(config.unstableFeatureVersionsEnabled),
        config.migrationEnabled,
        incarnationId,
        listenerInfo)

      // Add the registration manager to the list of metadata publishers, so that it receives
      // callbacks when the cluster registrations change.
      metadataPublishers.add(registrationManager)

      // Set up the dynamic config publisher. This runs even in combined mode, since the broker
      // has its own separate dynamic configuration object.
      metadataPublishers.add(new DynamicConfigPublisher(
        config,
        sharedServer.metadataPublishingFaultHandler,
        immutable.Map[String, ConfigHandler](
          // controllers don't host topics, so no need to do anything with dynamic topic config changes here
          ConfigType.BROKER -> new BrokerConfigHandler(config, quotaManagers)
        ),
        "controller"))

      // Register this instance for dynamic config changes to the KafkaConfig. This must be called
      // after the authorizer and quotaManagers are initialized, since it references those objects.
      // It must be called before DynamicClientQuotaPublisher is installed, since otherwise we may
      // miss the initial update which establishes the dynamic configurations that are in effect on
      // startup.
      config.dynamicConfig.addReconfigurables(this)

      // Set up the client quotas publisher. This will enable controller mutation quotas and any
      // other quotas which are applicable.
      metadataPublishers.add(new DynamicClientQuotaPublisher(
        config,
        sharedServer.metadataPublishingFaultHandler,
        "controller",
        clientQuotaMetadataManager))

      // Set up the SCRAM publisher.
      metadataPublishers.add(new ScramPublisher(
        config,
        sharedServer.metadataPublishingFaultHandler,
        "controller",
        credentialProvider
      ))

      // Set up the DelegationToken publisher.
      // We need a tokenManager for the Publisher
      // The tokenCache in the tokenManager is the same used in DelegationTokenControlManager
      metadataPublishers.add(new DelegationTokenPublisher(
          config,
          sharedServer.metadataPublishingFaultHandler,
          "controller",
          new DelegationTokenManager(config, tokenCache, time)
      ))

      // Set up the metrics publisher.
      metadataPublishers.add(new ControllerMetadataMetricsPublisher(
        sharedServer.controllerServerMetrics,
        sharedServer.metadataPublishingFaultHandler
      ))

      // Set up the ACL publisher.
      metadataPublishers.add(new AclPublisher(
        config.nodeId,
        sharedServer.metadataPublishingFaultHandler,
        "controller",
        authorizer
      ))

      // Install all metadata publishers.
      FutureUtils.waitWithLogging(logger.underlying, logIdent,
        "the controller metadata publishers to be installed",
        sharedServer.loader.installPublishers(metadataPublishers), startupDeadline, time)

      val authorizerFutures: Map[Endpoint, CompletableFuture[Void]] = endpointReadyFutures.futures().asScala.toMap

      /**
       * Enable the controller endpoint(s). If we are using an authorizer which stores
       * ACLs in the metadata log, such as StandardAuthorizer, we will be able to start
       * accepting requests from principals included super.users right after this point,
       * but we will not be able to process requests from non-superusers until AclPublisher
       * publishes metadata from the QuorumController. MetadataPublishers do not publish
       * metadata until the controller has caught up to the high watermark.
       */
      val socketServerFuture = socketServer.enableRequestProcessing(authorizerFutures)

      //用于 broker 和 controller 通信
      val controllerNodeProvider = RaftControllerNodeProvider(raftManager, config)
      registrationChannelManager = new NodeToControllerChannelManagerImpl(
        controllerNodeProvider,
        time,
        metrics,
        config,
        "registration",
        s"controller-${config.nodeId}-",
        5000)
      registrationChannelManager.start()
      registrationManager.start(registrationChannelManager)

      // Block here until all the authorizer futures are complete
      FutureUtils.waitWithLogging(logger.underlying, logIdent,
        "all of the authorizer futures to be completed",
        CompletableFuture.allOf(authorizerFutures.values.toSeq: _*), startupDeadline, time)

      // Wait for all the SocketServer ports to be open, and the Acceptors to be started.
      FutureUtils.waitWithLogging(logger.underlying, logIdent,
        "all of the SocketServer Acceptors to be started",
        socketServerFuture, startupDeadline, time)
    } catch {
      case e: Throwable =>
        maybeChangeStatus(STARTING, STARTED)
        sharedServer.controllerStartupFaultHandler.handleFault("caught exception", e)
        shutdown()
        throw e
    }
  }
```

总结一下比较重要的几点:

1. 创建了能够参与选举 (raft 算法中的选举) 的 quorumController 
2. 创建了控制的路由组件 (我随便起的名字), 将来自客户端的请求交给不同的 hander 来处理
3. 使用线程池来处理上述请求
4. 创建了 broker 和 controller 之间通信的管道
