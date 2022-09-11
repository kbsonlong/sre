# S2I自定义构建器和模板


### 准备工作
#### S2I自定义镜像构建器
● assemble（必需）：从源代码构建应用程序制品的脚本 assemble。
● run（必需）：用于运行应用程序的脚本。
● save-artifacts（可选）：管理增量构建过程中的所有依赖。
● usage（可选）：提供说明的脚本。
● test （可选）：用于测试的脚本。

### 创建镜像构建器

#### 准备S2I目录

1. 安装s2i

```bash
wget https://github.com/openshift/source-to-image/releases/download/v1.2.04/source-to-image-v1.1.14-874754de-linux-386.tar.gz
 tar -xvf source-to-image-v1.1.14-874754de-linux-386.tar.gz
 ls
s2i source-to-image-v1.1.14-874754de-linux-386.tar.gz  sti
 cp s2i /usr/local/bin
 ```

2. 初始化镜像构建器

```bash
s2i create java-ubuntu22 java-ubuntu22
cd java-ubuntu22
```

3. 目录结构初始化

```bash
.
├── Dockerfile
├── Makefile
├── README.md
├── prometheus-config.yml
├── s2i
│   └── bin
│       ├── assemble
│       ├── run
│       ├── save-artifacts
│       └── usage
└── test
    ├── run
    └── test-app
        ├── b2i-jar-java8.jar  ## 默认不存在
        └── index.html

4 directories, 11 files
```

### 修改Dockerfile

```dockerfile
# java-ubuntu22
FROM ubuntu:22.04 

# TODO: Put the maintainer name in the image metadata
# LABEL maintainer="Your Name <your@email.com>"

# TODO: Rename the builder environment variable to inform users about application you provide them
# ENV BUILDER_VERSION 1.0

# TODO: Set labels used in OpenShift to describe the builder image
LABEL io.k8s.description="Java 8 web application" \
    io.k8s.display-name="Java 8 Web" \
    io.openshift.expose-services="8080:http" \
    io.openshift.tags="builder,java,web" \
    # this label tells s2i where to find its mandatory scripts
    # (run, assemble, save-artifacts)
    io.openshift.s2i.scripts-url="image:///usr/libexec/s2i"

# TODO: Install required packages here:
# RUN yum install -y ... && yum clean all -y
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    openjdk-8-jdk && \
    rm -rf /var/lib/apt/lists

WORKDIR /opt

# TODO (optional): Copy the builder files into /opt/app-root
# COPY ./<builder_folder>/ /opt/app-root/

# TODO: Copy the S2I scripts to /usr/libexec/s2i, since openshift/base-centos7 image
# sets io.openshift.s2i.scripts-url label that way, or update that label
COPY ./s2i/bin/ /usr/libexec/s2i

# TODO: Drop the root user and make the content of /opt/app-root owned by user 1001
# RUN chown -R 1001:1001 /opt/app-root
RUN chgrp -R 0 /usr/libexec/s2i \
  && chmod -R u=rwx,go=rx /usr/libexec/s2i && \
  chgrp -R 0 /opt \
  && chmod -R u=rwx,go=rx /opt && \
  chown -R 1001:1001 /opt && \
  chown -R 1001:1001 /usr/libexec/s2i/assemble

RUN mkdir -p /opt/prometheus/etc \
  && curl https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.17.0/jmx_prometheus_javaagent-0.17.0.jar \
          -o /opt/prometheus/jmx_prometheus_javaagent.jar
COPY prometheus-config.yml /opt/prometheus/etc 

# This default user is created in the openshift/base-centos7 image
USER 1001

# TODO: Set the default port for applications built using this image
EXPOSE 8080

# TODO: Set the default CMD for the image
CMD ["/usr/libexec/s2i/usage"]
```

### 修改S2I脚本

#### assemble

```bash
#!/bin/bash -e
#
# S2I assemble script for the 'java-ubuntu22' image.
# The 'assemble' script builds your application source so that it is ready to run.
#
# For more information refer to the documentation:
#	https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

# If the 'java-ubuntu22' assemble script is executed with the '-h' flag, print the usage.
if [[ "$1" == "-h" ]]; then
	exec /usr/libexec/s2i/usage
fi

# Restore artifacts from the previous build (if they exist).
#
echo "---> Installing application..."
mkdir -p /opt/app
ls /tmp/src/*
mv /tmp/src/* /opt/app/
chmod +x /opt/app/*.jar
```

> 默认情况下，s2i build将应用程序源代码放在/tmp/src。上述命令将应用程序jar包复制到/opt/app/目录下

#### run

```bash
#!/bin/bash -e
#
# S2I run script for the 'java-ubuntu22' image.
# The run script executes the server that runs your application.
#
# For more information see the documentation:
#	https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

PROMETHEUS_JMX_OPTS="-javaagent:/opt/prometheus/jmx_prometheus_javaagent.jar=${PROMETHEUS_JMX_PORT}:/opt/prometheus/etc/prometheus-config.yml"

# Always include jolokia-opts, which can be empty if switched off via env
JAVA_OPTIONS="${JAVA_OPTIONS:+${JAVA_OPTIONS} }"

# Temporary options variable until the harmonization hawt-app PR #5 has been applied (hopefully)
JVM_ARGS="${JVM_ARGS:+${JVM_ARGS} }${JAVA_OPTIONS} ${PROMETHEUS_JMX_OPTS}"
export JAVA_OPTIONS JVM_ARGS PROMETHEUS_JMX_OPTS

exec java -jar ${JVM_ARGS} /opt/app/app.jar
```

### 构建与运行

#### 创建镜像构建器

```bash
make build 
```

#### 使用镜像构建器创建应用程序镜像

```bash
# s2i build ./test/test-app  java-ubuntu22 sample-app                           
---> Installing application...
/tmp/src/b2i-jar-java8.jar
/tmp/src/index.html
Build completed successfully
```

> 按照assemble脚本定义的逻辑，S2I使用镜像构建器作为基础创建应用程序镜像，并从test/test-app目录注入源代码。

#### 测试运行应用程序镜像

```bash
 # docker run -p 8080:8080 -p 12345:12345 -e PROMETHEUS_JMX_PORT=12345  sample-app

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::  (v1.4.1.BUILD-SNAPSHOT)

2022-07-28 07:51:48.829  INFO 1 --- [           main] io.kubesphere.devops.Application         : Starting Application v0.0.1-SNAPSHOT on 322634c30cc2 with PID 1 (/opt/app/app.jar started by ? in /opt)
2022-07-28 07:51:48.839  INFO 1 --- [           main] io.kubesphere.devops.Application         : No active profile set, falling back to default profiles: default
2022-07-28 07:51:48.906  INFO 1 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@5ae50ce6: startup date [Thu Jul 28 07:51:48 GMT 2022]; root of context hierarchy
2022-07-28 07:51:49.694  INFO 1 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8080 (http)
2022-07-28 07:51:49.700  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service Tomcat
2022-07-28 07:51:49.701  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.5
2022-07-28 07:51:49.749  INFO 1 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-28 07:51:49.749  INFO 1 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 845 ms
2022-07-28 07:51:49.820  INFO 1 --- [ost-startStop-1] o.s.b.w.servlet.ServletRegistrationBean  : Mapping servlet: 'dispatcherServlet' to [/]
2022-07-28 07:51:49.822  INFO 1 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'characterEncodingFilter' to: [/*]
2022-07-28 07:51:49.822  INFO 1 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
2022-07-28 07:51:49.823  INFO 1 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'httpPutFormContentFilter' to: [/*]
2022-07-28 07:51:49.823  INFO 1 --- [ost-startStop-1] o.s.b.w.servlet.FilterRegistrationBean   : Mapping filter: 'requestContextFilter' to: [/*]
2022-07-28 07:51:49.988  INFO 1 --- [           main] s.w.s.m.m.a.RequestMappingHandlerAdapter : Looking for @ControllerAdvice: org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@5ae50ce6: startup date [Thu Jul 28 07:51:48 GMT 2022]; root of context hierarchy
2022-07-28 07:51:50.047  INFO 1 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/]}" onto public java.lang.String io.kubesphere.devops.HelloWorldController.sayHello()
2022-07-28 07:51:50.050  INFO 1 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
2022-07-28 07:51:50.050  INFO 1 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/error]}" onto public org.springframework.http.ResponseEntity<java.util.Map<java.lang.String, java.lang.Object>> org.springframework.boot.autoconfigure.web.BasicErrorController.error(javax.servlet.http.HttpServletRequest)
2022-07-28 07:51:50.063  INFO 1 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/webjars/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2022-07-28 07:51:50.063  INFO 1 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2022-07-28 07:51:50.081  INFO 1 --- [           main] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped URL path [/**/favicon.ico] onto handler of type [class org.springframework.web.servlet.resource.ResourceHttpRequestHandler]
2022-07-28 07:51:50.153  INFO 1 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2022-07-28 07:51:50.176  INFO 1 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080 (http)
2022-07-28 07:51:50.178  INFO 1 --- [           main] io.kubesphere.devops.Application         : Started Application in 1.6 seconds (JVM running for 2.063)
```

> 8080是java程序定义的端口，12345是jmx_prometheus_agent定义的端口

```bash
# curl http://localhost:12345|more
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11008  100 11008    0     0  54117      0 --:--:-- --:--:-- --:--:-- 56163
# HELP jvm_memory_pool_allocated_bytes_total Total bytes allocated in a given JVM memory pool. Only updated after GC, not continuously.
# TYPE jvm_memory_pool_allocated_bytes_total counter
jvm_memory_pool_allocated_bytes_total{pool="Code Cache",} 1.1579328E7
jvm_memory_pool_allocated_bytes_total{pool="PS Eden Space",} 5.42141224E8
jvm_memory_pool_allocated_bytes_total{pool="PS Old Gen",} 1.2845896E7
jvm_memory_pool_allocated_bytes_total{pool="PS Survivor Space",} 2.0936672E7
jvm_memory_pool_allocated_bytes_total{pool="Compressed Class Space",} 3700680.0
jvm_memory_pool_allocated_bytes_total{pool="Metaspace",} 3.0138704E7
# HELP jvm_threads_current Current thread count of a JVM
# TYPE jvm_threads_current gauge
jvm_threads_current 17.0
# HELP jvm_threads_daemon Daemon thread count of a JVM
# TYPE jvm_threads_daemon gauge
jvm_threads_daemon 15.0
# HELP jvm_threads_peak Peak thread count of a JVM
# TYPE jvm_threads_peak gauge
jvm_threads_peak 17.0
# HELP jvm_threads_started_total Started thread count of a JVM
# TYPE jvm_threads_started_total counter
jvm_threads_started_total 20.0
# HELP jvm_threads_deadlocked Cycles of JVM-threads that are in deadlock waiting to acquire object monitors or ownable synchronizers
# TYPE jvm_threads_deadlocked gauge
jvm_threads_deadlocked 0.0
# HELP jvm_threads_deadlocked_monitor Cycles of JVM-threads that are in deadlock waiting to acquire object monitors
# TYPE jvm_threads_deadlocked_monitor gauge
jvm_threads_deadlocked_monitor 0.0
# HELP jvm_threads_state Current count of threads by state
# TYPE jvm_threads_state gauge
jvm_threads_state{state="NEW",} 0.0
jvm_threads_state{state="TIMED_WAITING",} 7.0
jvm_threads_state{state="BLOCKED",} 0.0
jvm_threads_state{state="RUNNABLE",} 8.0
jvm_threads_state{state="WAITING",} 2.0
jvm_threads_state{state="TERMINATED",} 0.0
# HELP jvm_classes_currently_loaded The number of classes that are currently loaded in the JVM
# TYPE jvm_classes_currently_loaded gauge
jvm_classes_currently_loaded 5736.0
# HELP jvm_classes_loaded_total The total number of classes that have been loaded since the JVM has started execution
# TYPE jvm_classes_loaded_total counter
jvm_classes_loaded_total 5736.0
# HELP jvm_classes_unloaded_total The total number of classes that have been unloaded since the JVM has started execution
# curl http://localhost:8080      
Hello,World!%    
```

### 自定义S2I模板
```yaml
apiVersion: devops.kubesphere.io/v1alpha1
kind: S2iBuilderTemplate
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
    builder-type.kubesphere.io/s2i: "s2i"
  name: java-ubuntu
spec:
  containerInfo:
    - builderImage: java-ubuntu22  ## 自定义的镜像构建器镜像
  codeFramework: java # type of code framework
  defaultBaseImage: java-ubuntu22 # default Image Builder (can be replaced by a customized image)
  version: 0.0.1 # Builder template version
  description: "模板描述" # Builder template description
```

```bash
kubectl apply -f s2ibuildertemplate.yaml
```

| 标签名称 | 选项 | 定义 |
| --- | --- | --- |
| builder-type.kubesphere.io/s2i | "s2i" | 模板类型为 S2I，基于应用程序源代码构建镜像。|
| builder-type.kubesphere.io/b2i |	"b2i" | 模板类型为 B2I，基于二进制文件或其他制品构建镜像。 |
| binary-type.kubesphere.io | "jar","war","binary" | 该类型为 B2I 类型的补充，在选择 B2I 类型时需要。例如，当提供 Jar 包时，选择 "jar" 类型。在 KubeSphere v2.1.1 及更高版本，允许自定义 B2I 模板。|


#### 官方demo
```yaml
apiVersion: devops.kubesphere.io/v1alpha1
kind: S2iBuilderTemplate
metadata:
  annotations:
    descriptionCN: Java 应用的构建器模版。通过该模版可构建出直接运行的应用镜像。
    descriptionEN: This is a builder template for Java builds whose result can be
      run directly without any further application server.It's suited ideally for
      microservices with a flat classpath (including "far jars").
    devops.kubesphere.io/s2i-template-url: https://github.com/kubesphere/s2i-java-container/blob/master/java/images
    helm.sh/hook: pre-install
  labels:
    binary-type.kubesphere.io: jar
    builder-type.kubesphere.io/b2i: b2i
    builder-type.kubesphere.io/s2i: s2i
    controller-tools.k8s.io: "1.0"
  name: java
spec:
  codeFramework: java
  containerInfo:
  - buildVolumes:
    - s2i_java_cache:/tmp/artifacts
    builderImage: kubesphere/java-8-centos7:v3.2.0
    runtimeArtifacts:
    - source: /deployments
    runtimeImage: kubesphere/java-8-runtime:v3.2.0
  - buildVolumes:
    - s2i_java_cache:/tmp/artifacts
    builderImage: kubesphere/java-11-centos7:v3.2.0
    runtimeArtifacts:
    - source: /deployments
    runtimeImage: kubesphere/java-11-runtime:v3.2.0
  defaultBaseImage: kubesphere/java-8-centos7:v3.2.0
  description: This is a builder template for Java builds whose result can be run
    directly without any further application server.It's suited ideally for microservices
    with a flat classpath (including "far jars")
  environment:
  - defaultValue: ""
    description: Arguments to use when calling Maven, replacing the default package
      hawt-app:build -DskipTests -e. Please be sure to run the hawt-app:build goal
      (when not already bound to the package execution phase), otherwise the startup
      scripts won't work.
    key: MAVEN_ARGS
    required: false
    type: string
  - defaultValue: ""
    description: Additional Maven arguments, useful for temporary adding arguments
      like -X or -am -pl .
    key: MAVEN_ARGS_APPEND
    required: false
    type: string
  - defaultValue: ""
    description: With Repositories you specify from which locations you want to download
      certain artifacts, such as dependencies and maven-plugins.
    key: MAVEN_MIRROR_URL
    required: false
    type: string
  - defaultValue: ""
    description: If set then the Maven repository is removed after the artifact is
      built. This is useful for keeping the created application image small, but prevents
      incremental builds. The default is false
    key: MAVEN_CLEAR_REPO
    required: false
    type: boolean
  - defaultValue: ""
    description: Path to target/ where the jar files are created for multi module
      builds. These are added to ${MAVEN_ARGS}
    key: ARTIFACT_DIR
    required: false
    type: string
  - defaultValue: ""
    description: Arguments to use when copying artifacts from the output dir to the
      application dir. Useful to specify which artifacts will be part of the image.
      It defaults to -r hawt-app/* when a hawt-app dir is found on the build directory,
      otherwise jar files only will be included (*.jar).
    key: ARTIFACT_COPY_ARGS
    required: false
    type: string
  - defaultValue: ""
    description: the directory where the application resides. All paths in your application
      are relative to this directory. By default it is the same directory where this
      startup script resides.
    key: JAVA_APP_DIR
    required: false
    type: string
  - defaultValue: ""
    description: directory holding the Java jar files as well an optional classpath
      file which holds the classpath. Either as a single line classpath (colon separated)
      or with jar files listed line-by-line. If not set JAVA_LIB_DIR is the same as
      JAVA_APP_DIR.
    key: JAVA_LIB_DIR
    required: false
    type: string
  - defaultValue: ""
    description: options to add when calling java
    key: JAVA_OPTIONS
    required: false
    type: string
  - defaultValue: ""
    description: a number >= 7. If the version is set then only options suitable for
      this version are used. When set to 7 options known only to Java > 8 will be
      removed. For versions >= 10 no explicit memory limit is calculated since Java
      >= 10 has support for container limits.
    key: JAVA_MAJOR_VERSION
    required: false
    type: string
  - defaultValue: ""
    description: is used when no -Xmx option is given in JAVA_OPTIONS. This is used
      to calculate a default maximal Heap Memory based on a containers restriction.
      If used in a Docker container without any memory constraints for the container
      then this option has no effect. If there is a memory constraint then -Xmx is
      set to a ratio of the container available memory as set here. The default is
      25 when the maximum amount of memory available to the container is below 300M,
      50 otherwise, which means in that case that 50% of the available memory is used
      as an upper boundary. You can skip this mechanism by setting this value to 0
      in which case no -Xmx option is added.
    key: JAVA_MAX_MEM_RATIO
    required: false
    type: string
  - defaultValue: ""
    description: is used when no -Xms option is given in JAVA_OPTIONS. This is used
      to calculate a default initial Heap Memory based on a containers restriction.
      If used in a Docker container without any memory constraints for the container
      then this option has no effect. If there is a memory constraint then -Xms is
      set to a ratio of the container available memory as set here. By default this
      value is not set.
    key: JAVA_INIT_MEM_RATIO
    required: false
    type: string
  - defaultValue: ""
    description: restrict manually the number of cores available which is used for
      calculating certain defaults like the number of garbage collector threads. If
      set to 0 no base JVM tuning based on the number of cores is performed.
    key: JAVA_MAX_CORE
    required: false
    type: string
  - defaultValue: ""
    description: set this to get some diagnostics information to standard out when
      things are happening
    key: JAVA_DIAGNOSTICS
    required: false
    type: string
  - defaultValue: ""
    description: main class to use as argument for java. When this environment variable
      is given, all jar files in $JAVA_APP_DIR are added to the classpath as well
      as $JAVA_LIB_DIR.
    key: JAVA_MAIN_CLASS
    required: false
    type: string
  - defaultValue: ""
    description: A jar file with an appropriate manifest so that it can be started
      with java -jar if no $JAVA_MAIN_CLASS is set. In all cases this jar file is
      added to the classpath, too.
    key: JAVA_APP_JAR
    required: false
    type: string
  - defaultValue: ""
    description: Name to use for the process
    key: JAVA_APP_NAME
    required: false
    type: string
  - defaultValue: ""
    description: the classpath to use. If not given, the startup script checks for
      a file ${JAVA_APP_DIR}/classpath and use its content literally as classpath.
      If this file doesn't exists all jars in the app dir are added (classes:${JAVA_APP_DIR}/*).
    key: JAVA_CLASSPATH
    required: false
    type: string
  - defaultValue: ""
    description: If set remote debugging will be switched on
    key: JAVA_DEBUG
    required: false
    type: string
  - defaultValue: ""
    description: If set enables suspend mode in remote debugging
    key: JAVA_DEBUG_SUSPEND
    required: false
    type: string
  - defaultValue: ""
    description: 'Port used for remote debugging. Default: 5005'
    key: JAVA_DEBUG_PORT
    required: false
    type: string
  - defaultValue: ""
    description: The URL of the proxy server that translates into the http.proxyHost
      and http.proxyPort system properties.
    key: HTTP_PROXY
    required: false
    type: string
  - defaultValue: ""
    description: The URL of the proxy server that translates into the https.proxyHost
      and https.proxyPort system properties.
    key: HTTPS_PROXY
    required: false
    type: string
  - defaultValue: ""
    description: The list of hosts that should be reached directly, bypassing the
      proxy, that translates into the http.nonProxyHosts system property.
    key: NO_PROXY
    required: false
    type: string
  iconPath: assets/java.png
  version: 0.0.2
  ```

### 参考资料

[s2i-base-container](https://github.com/kubesphere/s2i-base-container)

[s2i-java-container](https://github.com/kubesphere/s2i-java-container)

[s2i-java-runtimeImage](https://github.com/kubesphere/s2i-java-runtimeImage)

