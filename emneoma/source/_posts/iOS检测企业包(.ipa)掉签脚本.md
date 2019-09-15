---
title: iOS检测企业包(.ipa)掉签脚本
date: 2019-09-14 22:26:05
tags: [iOS,企业签名,shell]
Categories: shell
---

{% cq %}
“客户装不上app啦！”
“客户打开app闪退啦！”
“ios检查一下是不是掉签啦！”
{% endcq %}

<!-- more -->

<div class="note info"><p>遇到企业证书被Revoke的情况，会导致已装好并且手动信任的企业包app打开闪退，或新用户装上app图标是灰色的。</p></div>


太烦了，每次检查要重新下载安装一次确认，浪费时间，流量，又没有技术含量，不能每次都要因为这种问题浪费宝贵的时间，而且是否掉签我们作为开发要用事实&数据说话！

- 我们可以用Apple自带的codesign命令对app文件内的签名导出
- 再通过OpenSSL命令去检查证书的有效期

如果检查不到有效期或者显示了"Revoked"，自然表示签名不可用，也意味着证书挂了。
参考了几篇文（找不到了），并弄了一个脚本：

```shell
# USAGE: ./checkSign.sh {ipa文件名：xxx.ipa}
# 看到⚠️证明该ipa证书签名有问题


# 删除文件
echo "🗑 删除codesign,pem,plist,和“Payload”文件夹及其子文件"
if [ -d "Payload" ] ; then
rm -rf Payload
fi
rm -f codesign0
rm -f codesign1
rm -f codesign2
rm -f *.pem
rm -f *.plist
rm -f iTunesArtwork


# 参数检查
canshu=$1
ipaFile=${canshu##*/}
# 截取&拼接.app
appFile=${ipaFile%.*}".app"
islegal=$(echo $1 | grep ".ipa$")
echo "👀 请检查参数"
echo "ipa: $ipaFile"
echo "app: $appFile"
if [ -n "$islegal" ]; then
	if [ ! -f "$ipaFile" ]; then
		echo "❌ “$1”文件不存在，结束进程"
		exit 0
	fi
else
	echo "❌ 参数格式必须是文件全名:xxx.ipa，结束进程"
	exit 0
fi


# 解压
unzip -q $ipaFile
# 导出签名
codesign -dvv --extract-certificates Payload/*.app
if [ ! -f "codesign0" ] || [ ! -f "codesign1" ] || [ ! -f "codesign2" ]; then
	echo "⚠️ 导出codesign完整3个文件失败，该ipa文件证书过期或者被损坏"
	#TODO: 可在这里做替换ipa操作
else
	openssl x509 c-inform DER -in codesign0 -out codesign0.pem
	openssl x509 -inform DER -in codesign1 -out codesign1.pem
	openssl x509 -inform DER -in codesign2 -out codesign2.pem
	cat codesign1.pem codesign2.pem > cachain.pem
	openssl x509 -inform DER -in codesign0 -noout -nameopt -oneline -subject -serial -dates
	if openssl ocsp -issuer cachain.pem -cert codesign0.pem -url `openssl x509 -in codesign0.pem -noout -ocsp_uri` -CAfile cachain.pem -header 'host' 'ocsp.apple.com' | grep revoked ; then
	    echo "⚠️ 证书被Revoke，请替换"
	    #TODO: 可在这里做替换ipa操作
	else
	    echo "💯执行完毕，该ipa包证书可用"
	fi
fi

```

这个脚本可以丢给运维，让他做定时检查，他还可以做掉签报警和切换等..
以后掉签问题问运维，不要再来问我啦～（别打扰我摸鱼）