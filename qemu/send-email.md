## 一、安装依赖

```bash
yum install -y git-email
yum install -y perl-Mail*
```

## 二、准备 patch 文件

## 三、邮箱开启 `smtp` 服务，获取 smtp 授权密码

配置 git config

```
git config --global sendemail.smtpserver smtp.qq.com
git config --global sendemail.smtpuser sunnyzhyy@qq.com
git config --global sendemail.smtppass xxxx[smtp授权码]
git config --global sendemail.smtpserverpor 465
git config --global sendemail.smtpencryption ssl
```

!> 我这里使用的是 qq 邮箱，其 smtp 默认只支持 `ssl` 加密，我先是配置成 `tls` 就一直发送邮件失败，改成 `ssl` 才能发送成功。

## 四、使用 git send-email 发送 patch

```bash
git send-email --to=qemu-devel@nongnu.org /home/your.patch
```

