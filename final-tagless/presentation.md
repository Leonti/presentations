class: center, middle

# Final Tagless

---
class: middle
# Kafka SSM Security
Case study of Final Tagless

---
class: middle

# Apache Kafka
Open-source pub/sub framework developed at LinkedIn  
Similar to Kinesis but:  
- can have unlimited retention of messages
- high throughput
- written in Scala
- self-managed :(
- limited tooling :( 
---
class: middle, wide-content

# What was available

```bash
# create a topic
bin/kafka-topics.sh --create --zookeeper localhost:2181 \
--replication-factor 1 --partitions 1 --topic test

# create a user
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config \
'SCRAM-SHA-512=[password=alice-secret]' --entity-type users --entity-name Alice

# give user permissions to a topic
kafka-acls --authorizer-properties zookeeper.connect=localhost:2181 \
--add --allow-principal User:Alice --operation All --topic '*' --cluster
```
- tedious  
- insecure  
- error-prone  
- non-reproducible  
---
class: middle

# What we wanted

```yaml
TestTopic:
Type: AWS::SSM::Parameter
Properties:
    Type: String
    Name: /kafka-security/example-cluster/topics/test-topic
    Value: 1,10,24

TestUserAcl:
Type: AWS::SSM::Parameter
Properties:
    Name: /kafka-security/example-cluster/users/test-user
    Type: String
    Value: |
    Topic,LITERAL,test-topic,Read,Allow,*,
    Group,LITERAL,test-partition,Write,Allow,*
```
- declarative  
- secure  
  - no SSH access needed  
  - auditable  
- setup is in source control    
---
class: center, middle

# Kafka SSM Security

---

background-image: url('images/kss-diagram.svg')

---

background-image: url('images/ports-and-adaptors.svg')

---

class: center, middle

# Testing Approach

---

class: middle

## Inputs:  
 - SSM Parameter Store
 - Kafka Topics
 - Kafka Users
 - Kafka ACLs
 
## Outputs:
 - Kafka Topics
 - Kafka Users
 - Kafka ACLs
 - Metrics
 - Logs

---

class: center, middle

# Unit is a module, not a class

---
class: center, middle

# Integration tests for thin IO implementations  
# Unit tests for everything else 

---
class: center, middle

# State implementations for testing, IO for the real thing

---

background-image: url('images/sync-test.svg')

---

class: middle, wide-content

```scala
  it should "log errors for failed topics" in {

    val kafkaTopics = new KafkaTopicsAlgState(Set("a-topic"))
    val ssm = new SsmAlgTest[TestProgram](
      SsmParameter("/kafka-security/ci-cluster/topics/unparseable-topic", 
        "unparseable"),
      SsmParameter("/kafka-security/ci-cluster/topics/test-topic", "1,10,24")
    )

    val topicSync = new TopicSync("ci-cluster", kafkaTopics, ssm, 
        new MetricsAlgState(), new LogAlgState())
    val (state, _) = topicSync.sync.run(SystemState()).value

    state.topics shouldBe 
      List(Topic("test-topic", ReplicationFactor(1), 
        PartitionCount(10), Some(RetentionHs(24))))
    state.errors.length shouldBe 1
    state.metricsSent.toSet shouldBe 
        Set(TopicsCreated(1), TopicsFailed(1), TopicsOutOfSync(0))
  }
```

---

class: middle, wide-content

# Inputs to the system
```scala
    val kafkaTopics = new KafkaTopicsAlgState(Set("a-topic"))
    val ssm = new SsmAlgTest[TestProgram](
      SsmParameter("/kafka-security/ci-cluster/topics/unparseable-topic", 
        "unparseable"),
      SsmParameter("/kafka-security/ci-cluster/topics/test-topic", "1,10,24")
    )
```

---

class: middle, wide-content

# Outputs
```scala
    val topicSync = new TopicSync("ci-cluster", kafkaTopics, ssm, 
        new MetricsAlgState(), new LogAlgState())
    val (state, _) = topicSync.sync.run(SystemState()).value

    state.topics shouldBe 
      List(Topic("test-topic", ReplicationFactor(1), 
        PartitionCount(10), Some(RetentionHs(24))))
    state.errors.length shouldBe 1
    state.metricsSent.toSet shouldBe 
        Set(TopicsCreated(1), TopicsFailed(1), TopicsOutOfSync(0))
```

---
class: middle

# Separate problem description from interpretation
- Parameterize application with an effect  
- Use `IO` implementation/interpreter for the real app  
- Use `State` implementation for testing  

---

class: center, middle

# Final Tagless

---

class: middle, wide-content

# DSL
```scala
trait SsmAlg[F[_]] {
  def getParameterNames(filter: String): F[Set[String]]
  def getParameter(name: String): F[Option[SsmParameter]]
}

trait KafkaTopicsAlg[F[_]] {
  def createTopic(topic: Topic): F[Unit]
  def getTopics: F[Set[Topic]]
}

trait LogAlg[F[_]] {
  def info(message: String): F[Unit]
  def error(message: String): F[Unit]
}

trait MetricsAlg[F[_]] {
  def sendMetric(metric: Metric): F[Unit]
}
```
---

class: middle, wide-content

```scala
class TopicSync[F[_]: Monad](clusterName: String,
                             kafkaTopics: KafkaTopicsAlg[F],
                             ssm: SsmAlg[F],
                             metrics: MetricsAlg[F],
                             log: LogAlg[F]) {

  private def ssmConfig: SsmConfig[F] = new SsmConfig[F](clusterName, ssm)

  [... private functions]  

  val sync: F[Unit] = for {
    ssmResult <- ssmConfig.getTopics
    (ssmErrors, ssmTopics) = ssmResult
    _ <- ssmErrors.map(_.value).traverse(log.error)
    _ <- metrics.sendMetric(TopicsFailed(ssmErrors.length))
    existingTopics <- kafkaTopics.getTopics
    topicsToCreate = ssmTopics.filterNot(topic => 
        existingTopics.map(_.name).contains(topic.name))
    _ <- if (topicsToCreate.nonEmpty)
            log.info(s"Creating ${topicsToCreate.size} topics") 
         else
            log.info("No topics to create")
    _ <- [...creating topics]
    outOfSyncErrors = outOfSync(existingTopics, ssmTopics)
    _ <- outOfSyncErrors.toList.traverse(log.error)
    _ <- metrics.sendMetric(TopicsOutOfSync(outOfSyncErrors.size))
    _ <- metrics.sendMetric(TopicsCreated(topicsToCreate.size))
  } yield ()

```

---

class: center, middle

# Testing

---

class: middle, wide-content

```scala
case class SystemState(
                        usersAdded: List[User] = List(),
                        usersUpdated: List[User] = List(),
                        userNamesRemoved: List[UserName] = List(),
                        topics: List[Topic] = List(),
                        infos: List[String] = List(),
                        errors: List[String] = List(),
                        metricsSent: List[Metric] = List(),
                        aclsAdded: Map[Resource, Set[Acl]] = Map(),
                        aclsRemoved: Map[Resource, Set[Acl]] = Map())

type TestProgram[A] = State[SystemState, A]
```

---

class: middle, wide-content

```scala
class KafkaTopicsAlgState(topicNames: Set[String]) 
  extends KafkaTopicsAlg[TestProgram] {

override def createTopic(topic: Topic): TestProgram[Unit] = State { st =>
  (st.copy(topics = topic :: st.topics), ())
}

override def getTopics: TestProgram[Set[Topic]] = State.pure(topicNames
  .map(name => Topic(name, ReplicationFactor(1), PartitionCount(6), 
    Some(RetentionHs(24)))))
}
```
---

class: middle, wide-content

```scala
class MetricsAlgState extends MetricsAlg[TestProgram] {
  override def sendMetric(metric: Metric): TestProgram[Unit] = State { st =>
    (st.copy(metricsSent = metric :: st.metricsSent), ())
  }
}
```

---

class: middle, wide-content

```scala
  it should "log errors for failed topics" in {

    val kafkaTopics = new KafkaTopicsAlgState(Set("a-topic"))
    val ssm = new SsmAlgTest[TestProgram](
      SsmParameter("/kafka-security/ci-cluster/topics/unparseable-topic", 
        "unparseable"),
      SsmParameter("/kafka-security/ci-cluster/topics/test-topic", "1,10,24")
    )

    val topicSync = new TopicSync("ci-cluster", kafkaTopics, ssm, 
        new MetricsAlgState(), new LogAlgState())
    val (state, _) = topicSync.sync.run(SystemState()).value

    state.topics shouldBe 
      List(Topic("test-topic", ReplicationFactor(1), 
        PartitionCount(10), Some(RetentionHs(24))))
    state.errors.length shouldBe 1
    state.metricsSent.toSet shouldBe 
        Set(TopicsCreated(1), TopicsFailed(1), TopicsOutOfSync(0))
  }
```
---
class: middle, center

# IO implementations for real system
---
class: middle, wide-content

```scala
class KafkaTopics(zkClient: KafkaZkClient) extends KafkaTopicsAlg[IO] {
  val adminZkClient = new AdminZkClient(zkClient)

  override def createTopic(topic: Topic): IO[Unit] = IO {
    val topicProperties = new Properties
    topic.retentionHs.foreach(retentionHs =>
      topicProperties
        .setProperty("retention.ms", retentionHs.value.hours.toMillis.toString)
    )
    adminZkClient.createTopic(topic.name, topic.partitionCount.value, 
      topic.replicationFactor.value, topicProperties)
  }

  override def getTopics: IO[Set[Topic]] = IO {
    val configs = adminZkClient.getAllTopicConfigs()
    val assignments = zkClient
      .getPartitionAssignmentForTopics(configs.keys.toSet)

    configs.toList
      .flatMap({ case (topic, config) => assignments.get(topic).map(a => {
        val retentionHs = Option(config.getProperty("retention.ms"))
          .map(_.toLong.millis.toHours).map(RetentionHs.apply)
        Topic(topic, ReplicationFactor(a.head._2.size), 
          PartitionCount(a.size), retentionHs)
      })
    }).toSet
  }
}
```

---
class: middle, wide-content

```scala
val logger = new AWSLogger(Regions.AP_SOUTHEAST_2, logGroup, logStream)
val metrics = new CloudWatch(region, metricsNamespace)
val ssm = new SsmAws(region)

val zkClient = KafkaZkClient(zkEndpoint, JaasUtils.isZkSecurityEnabled, 
  30000, 30000, Int.MaxValue, Time.SYSTEM)

val topicSync = new TopicSync[IO](clusterName, new KafkaTopics(zkClient), 
  ssm, metrics, logger)
```
---
class: center, middle

# Integration Tests

---
class: middle, wide-content

```scala
it should "create a topic and read topic names" in {
  val kafkaZkClient = KafkaZkClient(zkEndpoint, JaasUtils.isZkSecurityEnabled, 
    30000, 30000, Int.MaxValue, Time.SYSTEM)

  val kafkaTopics = new KafkaTopics(kafkaZkClient)

  val topicName = s"topic-${randomUUID().toString}"
  kafkaTopics.createTopic(Topic(topicName, 
    ReplicationFactor(1), PartitionCount(1), Some(RetentionHs(1))))
    .unsafeRunSync()

  val topicNames = kafkaTopics.getTopics
    .map(_.map(_.name)).unsafeRunSync()

  topicNames should contain (topicName)
}
```

---
class: center, middle

# Limitations
---
class: middle

# Single monad
- `IO` for the real app
- `State` works reasonably well for testing

---
class: middle, wide-content

```scala
class SsmAlgTest[F[_]: Monad](parameters: SsmParameter*) extends SsmAlg[F] {

  private def map: Map[String, SsmParameter] = 
    parameters.map(p => p.name -> p).toMap

  override def getParameterNames(filter: String): F[Set[String]] = 
    Monad[F].pure(map.keys.toSet)

  override def getParameter(name: String): F[Option[SsmParameter]] = 
    Monad[F].pure(map.get(name))
}
```

```scala
val ssm = new SsmAlgTest[TestProgram]()
```

---
class: middle, wide-content

# When "`F[_]: Monad`" is not enough
FS2 requires "`F[_]: Sync`" instead of "`F[_]: Monad`"  
Testing monad needs to satisfy `Sync`  

```scala
  case class SystemState(
      tmpFiles: List[Path] = List(),
      localFilesRemoved: List[Path] = List(),
      localFilesWritten: List[(Path, String)] = List(),
      smbFilesWritten: List[(SmbFile, String)] = List(),
      s3FilesRead: List[S3File] = List()
  )
  type TestState[A] = StateT[IO, SystemState, A]
```
---
class: middle, center

# Thank You!

https://github.com/MYOB-Technology/kafka-ssm-security
---
https://softwaremill.com/free-tagless-compared-how-not-to-commit-to-monad-too-early/
https://www.becompany.ch/en/blog/2018/09/27/tagless-final-and-eff
