在prometheus中，告警管理由两部分构成:

- prometheus服务端通过配置告警规则产生告警，并负责将告警发送到AlertManager；
- AlertManager管理这些告警，按照策略处理告警（如忽略、聚合等），以及进行告警前转。

告警管理的设置的主要步骤包括：

- 启动并配置Alertmanager
- 配置prometheus server访问Alertmanager
- 在prometheus服务端配置告警规则

## Grouping 告警分组

Grouping可以将性质类似的警告分组成一个通知。如果当系统大规模故障时，如果不分组，将会瞬间产生大量告警，所有Grouping在此情况下很有用。

例如，对于成百上千的服务实例构成的集群下，一旦网络出错，很多服务无法访问数据库，prometheus将为每一个服务发送一个告警到Alertmanager。但是用户只想看到一个告警，而不是希望被成千上百的告警所困惑。所有Alertmanager可以配置通过告警的标签来进行分组聚合后前转。

告警分组，分组通知的时间和通知接受者都是需要在配置文件中确定。

## Inhibition 告警抑制

如果特定的告警已经发生，则某些告警需要被抑制。

例如，如果一整个集群不可访问的告警已经触发，则Alertmanager可以配置为任何后续该集群的告警都将没有动作。这样使得大量关于这个集群不可用产生的告警都经被阻止。

同样，告警抑制也可以通过Alertmanager配置文件来配置。

## Sliences 告警静默

Slience可以在给定的时间段内简单的忽略相关告警。slience基于匹配器工作，和路由树类似，检查告警是否和配置的正则表达式匹配，如果匹配就不会发送通知前转。

## Client Behavior 客户端行为

AlertManager对其客户端行为有特别的要求。这里不详述。
