## maven compile vs provided
### compile

This is the default scope, used if none is specified. Compile dependencies are available in all classpaths of a project. Furthermore, those dependencies are propagated to dependent projects.

### provided

This is much like compile, but indicates you expect the JDK or a container to provide the dependency at runtime. For example, when building a web application for the Java Enterprise Edition, you would set the dependency on the Servlet API and related Java EE APIs to scope provided because the web container provides those classes. This scope is only available on the compilation and test classpath, and is not transitive. 

### diff
- provided dependencies are not transitive (as you mentioned)
- provided scope is only available on the compilation and test classpath, whereas compile scope is available in all classpaths.
- provided dependencies are not packaged