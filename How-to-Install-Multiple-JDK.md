# How to install multiple JDK versions

### - Check the current JDK informaiton
`java -version ` *check java version*

` which java `  *check java jre path*

` cat ~/.bash_profile ` *check bashrc config*

` /usr/libexec/java_home -V  ` *check JDK local path*

### - Config Steps:
1. Install the JAVA JDK
  Download the JDK installer from https://www.oracle.com/java/technologies/javase-downloads.html, and install it.
  The JDK should be installed on the path: /Library/Java/JavaVirtualMachines/jdk1.xx.0.jdk
2. Add JAVA_HOME in .bashrc file.  
   **- open bashrc file**
   ` vim ~/.bash_profile ` 
   *add the JAVA_HOME:*
   ``` 
   export JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0.jdk/Contents/Home  
   export JAVA_11_HOME=/Library/Java/JavaVirtualMachines/jdk1.11.0.jdk/Contents/Home  
   export JAVA_HOME=$JAVA_8_HOME  
   ```
   **- open bashrc file**
   ` vim ~/.bash_profile ` 
