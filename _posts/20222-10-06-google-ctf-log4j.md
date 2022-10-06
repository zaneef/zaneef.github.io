---
title: Google CTF 2022 - Log4J
tags: [google ctf 2022, web] 
---

We can access the challenge with the link below

[https://log4j-web.2022.ctfcompetition.com/](https://log4j-web.2022.ctfcompetition.com/)

The application is called "Chatbot" and is based on a single text field that can receive commands, the backend interprets them, and the result is printed on the application frontend.

Luckly we have access to the application source code. The application file tree is the following.

```
.
├── chatbot
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── com
│       │   │       └── google
│       │   │           └── app
│       │   │               └── App.java
│       │   └── resources
│       │       └── log4j2.xml
│       └── test
│           └── java
│               └── com
│                   └── google
│                       └── app
│                           └── AppTest.java
├── Dockerfile
├── nsjail.cfg
├── server
│   ├── app.py
│   ├── requirements.txt
│   └── templates
│       └── index.html
└── start.sh
```

Let's take a look at the `App.java` file. A public class called `App` is defined and inside it is defined the `main` method. We can see that the flag is retrieved from an enviroment variable.

```java
public static void main(String[]args) {
  String flag = System.getenv("FLAG");
  if (flag == null || !flag.startsWith("CTF")) {
    LOGGER.error("{}", "Contact admin");
  }
  
  LOGGER.info("msg: {}", args);
  // TODO: implement bot commands
  String cmd = System.getProperty("cmd");
  if (cmd.equals("help")) {
    doHelp();
    return;
  }
  if (!cmd.startsWith("/")) {
    System.out.println("The command should start with a /.");
    return;
  }
  doCommand(cmd.substring(1), args);
}
```

Below the main method, we can see the definition of the application commands

```java
private static void doCommand(String cmd, String[] args) {
  switch(cmd) {
    case "help":
      doHelp();
      break;
    case "repeat":
      System.out.println(args[1]);
      break;
    case "time":
      DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/M/d H:m:s");
      System.out.println(dtf.format(LocalDateTime.now()));
      break;
    case "wc":
      if (args[1].isEmpty()) {
        System.out.println(0);
      } else {
        System.out.println(args[1].split(" ").length);
      }
      break;
    default:
      System.out.println("Sorry, you must be a premium member in order to run this command.");
  }
}
```

They can be described as follows

- `/help` print the help menu which contains all possible commands
- `/repeat` print the arguments
- `/time` print the current date from `LocalDateTime.now()`
- `/wc` print the number of words that follows. They must be separated by a space character

All over the App.java file we can see the usage of a particular class: [LogManager](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/LogManager.html). After searching it online we are able to see that is the "anchor point for Log4j logging system".

```java
public static Logger LOGGER = LogManager.getLogger(App.class);
```

Let's take a look at app.py. It's a standard Flask server definition. One function stands out

```python
def chat(cmd, text):
  # run java jar with a 10 second timeout
  res = subprocess.run(['java', '-jar', '-Dcmd=' + cmd, 'chatbot/target/app-1.0-SNAPSHOT.jar', '--', text], capture_output=True, timeout=10)
  print(res.stderr.decode('utf8'))
  return res.stdout.decode('utf-8')
```

That's pretty clear. The .jar chatbot application is executed as a new subprocess. 

Only one thing was not immediatly clear: `capture_output=True`. After searching it online I was able to find [this](https://stackoverflow.com/questions/41171791/how-to-suppress-or-capture-the-output-of-subprocess-run) StackOverflow thread in which is described as a method to capture both `STDERR` and `STOUT`. With it we can redirect all the subprocess' STDOUT to flask other than printing it on the console. STDERR, however, is printed on the console.  


The first thing I search is how to print enviroment variables with Log4j. We can do this with ${env:FLAG} but it's printed in the STDERR (thanks to lo4j2.xml file). We have to find a way to redirect the STDERR in the STDOUT.

We could try to send some Log4J commands that don't exists. Let's try with

```
%something
```

This will produce an exception and will be printed in the STDOUT.

We
