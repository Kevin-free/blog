# 网络代理导致的 git push 失败问题

## 开启网络自动代理 pac

当开启网络自动代理配置时，git，vscode 均 push 直接失败。

取消代理

取消全局代理：

```
git config --global http.proxy http://127.0.0.1:1080

git config --global https.proxy http://127.0.0.1:1080

git config --global --unset http.proxy

git config --global --unset https.proxy
```

git push 成功，vscode push 成功。
但有时会 timeout。

## 关闭网络自动代理

push 正常。
但是浏览器无法访问外网。

## 关闭网络自动代理，开启 chrome SwitchyOmega 插件代理时

push 正常，浏览器也可访问访问。

## 结论

设置代理 pac 模式时，会影响 git push。
使用 chrome 插件设置代理，可解决此问题。

## 参考

[Vscode 解决 Failed to connect to github.com port 443:connection timed out](https://blog.csdn.net/M_Edison/article/details/114186264)

[网络问题一次解决！(GitHub,Google)](https://www.daimajiaoliu.com/daima/6cc85bac3a66006)

[Chrome - SwitchyOmega 设置教程](https://manager.myxgj.com/device/chrome)
