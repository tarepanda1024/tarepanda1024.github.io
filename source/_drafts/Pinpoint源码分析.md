---
title: Pinpoint源码分析
tags:
---


启动类：
com.navercorp.pinpoint.bootstrap.PinpointBootStrap

```java
//获取javaagent的路径
        JavaAgentPathResolver javaAgentPathResolver = JavaAgentPathResolver.newJavaAgentPathResolver();
        String agentPath = javaAgentPathResolver.resolveJavaAgentPath();
        logger.info("JavaAgentPath:" + agentPath);
```

```java
final ClassPathResolver classPathResolver = new AgentDirBaseClassPathResolver(agentPath);
```

```java
    static final JarDescription bootstrap = new JarDescription("pinpoint-bootstrap", true);	
    public AgentDirectory resolve() {

        // find boot-strap.jar
        final String bootstrapJarName = this.findBootstrapJar(this.classPath);
        if (bootstrapJarName == null) {
            throw new IllegalStateException(bootstrap.getSimplePattern() + " not found.");
        }

        final String agentJarFullPath = parseAgentJarPath(classPath, bootstrapJarName);
        if (agentJarFullPath == null) {
            throw new IllegalStateException(bootstrap.getSimplePattern() + " not found. " + classPath);
        }
        final String agentDirPath = getAgentDirPath(agentJarFullPath);

        final BootDir bootDir = resolveBootDir(agentDirPath);

        final String agentLibPath = getAgentLibPath(agentDirPath);
        final List<URL> libs = resolveLib(agentLibPath);

        String agentPluginPath = getAgentPluginPath(agentDirPath);
        final List<String> plugins = resolvePlugins(agentPluginPath);

        final AgentDirectory agentDirectory = new AgentDirectory(bootstrapJarName, agentJarFullPath, agentDirPath, bootDir, libs, plugins);

        return agentDirectory;
    }
```