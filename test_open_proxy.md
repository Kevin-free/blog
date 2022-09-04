当设置网络代理时，git，vscode 均 push 失败。

取消代理

取消全局代理：

```
git config --global http.proxy http://127.0.0.1:1080

git config --global https.proxy http://127.0.0.1:1080

git config --global --unset http.proxy

git config --global --unset https.proxy
```

git push 成功，

