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

