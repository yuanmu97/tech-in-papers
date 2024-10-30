# 30-stateless
Source: https://www.redhat.com/en/topics/cloud-native-apps/stateful-vs-stateless




> stateless vs. stateful computing

* stateful computing 会存储过去的信息，但 stateless computing 不会
* stateful computing 会 track 一个用户的 session，包括过去的交互和 requests。日常的常见 app 都是 stateful 的
* stateless computing 一般只提供一个通用的功能，例如 CDN
* 区别：
  * 可扩展性：stateless 更好，stateful 需要更复杂的负载均衡以及 session 管理
  * 容错：stateless 天然地更好，因为不需要维护用户 session，而 stateful 则需要例如 session replication 等方法来实现容错
  * 资源效率：stateless 更好，stateful 需要额外的内存和算力维护 session 信息
  * 部署复杂性：stateless 更简单，stateful 需要管理多个 requests 的状态


