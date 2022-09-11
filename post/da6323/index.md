# kubebuilder开发operator


### 安装kubebuilder
```bash
brew install kubebuilder
kubebuilder version
```
<!--more-->
### 创建项目目录
```bash
mkdir custom-controllers
cd custom-controllers
go mod init controllers.alongparty.cn
```

### kubebuilder初始化项目
```bash
kubebuilder init --domain controller.alongparty.cn  --license apache2 --owner "kbsonlong" 
kubebuilder create api --group controller --version v1 --kind Application
make
```




### 参考资料
[使用kubebuilder开发operator详解](https://podsbook.com/posts/kubernetes/operator/)
