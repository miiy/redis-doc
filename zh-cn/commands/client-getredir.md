这个命令返回我们重定向我们的[追踪](/topics/client-side-caching)通知的客户端ID。当使用`CLIENT TRACKING`来启用追踪时，我们设置一个要重定向的客户端。然而，为了避免强制客户端库实现记住通知重定向到哪个ID，这个命令存在于以后能够改善自省，允许客户端以后检查重定向是否处于活动状态以及指向哪个客户端ID。