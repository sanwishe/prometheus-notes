amtool是一个能够与Alertmanager交互的命令行客户端。

## 使用

### 查看帮助：

```bash
# ./amtool -h
usage: amtool [<flags>] <command> [<args> ...]
 
View and modify the current Alertmanager state.
 
Config File: The alertmanager tool will read a config file in YAML format from one of two default config locations: $HOME/.config/amtool/config.yml or /etc/amtool/config.yml
 
All flags can be given in the config file, but the following are the suited for static configuration:
 
  alertmanager.url
        Set a default alertmanager url for each request
 
  author
        Set a default author value for new silences. If this argument is not
        specified then the username will be used
 
  require-comment
        Bool, whether to require a comment on silence creation. Defaults to true
 
  output
        Set a default output type. Options are (simple, extended, json)
 
  date.format
        Sets the output format for dates. Defaults to "2006-01-02 15:04:05 MST"
 
Flags:
  -h, --help           Show context-sensitive help (also try --help-long and --help-man).
      --date.format="2006-01-02 15:04:05 MST"
                       Format of date output
  -v, --verbose        Verbose running information
      --alertmanager.url=ALERTMANAGER.URL
                       Alertmanager to talk to
  -o, --output=simple  Output formatter (simple, extended, json)
      --version        Show application version.
 
Commands:
  help [<command>...]
  alert
    query* [<flags>] [<matcher-groups>...]
  silence
    add [<flags>] [<matcher-groups>...]
    expire [<silence-ids>...]
    import [<flags>] [<input-file>]
    query* [<flags>] [<matcher-groups>...]
    update [<flags>] [<update-ids>...]
  check-config [<check-files>...]
  config
```

### 工具配置

amtool进行告警查询和静默规则配置时，需要指定一些flags，包括Alertmanager的地址，认证信息，通知接受者等。这些flag可以在每次请求的时候带上，也可以以配置文件的方式写到固定路径(包括/etc/amtool/config.yml或者$HOME/.config/amtool/config.yml)。

文件格式如下：

```
# Define the path that amtool can find your `alertmanager` instance at
alertmanager.url: "http://localhost:9093"
 
# Override the default author. (unset defaults to your username)
author: me@example.com
 
# Force amtool to give you an error if you don't include a comment on a silence
comment_required: true
 
# Set a default output format. (unset defaults to simple)
output: extended
 
# Set a default receiver
receiver: team-X-pager
```

### 查看当前告警

```bash
# ./amtool alert
Alertname          Starts At                Summary
TooMuchGoroutines  2018-07-06 10:02:39 CST  too much goroutines of job prometheus.
TooMuchGoThreads   2018-07-06 11:05:26 CST  too much go thread of job prometheus
```

### 查询静默告警：

```bash
# ./amtool silence query
ID                                    Matchers                     Ends At                  Created By  Comment
d62a425b-d754-40b4-aab8-a2370e17d8a8  alertname=TooMuchGoroutines  2018-07-10 02:10:57 UTC  jmz         testing
```
