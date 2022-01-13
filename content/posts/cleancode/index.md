---
title: "浅谈代码优化"
date: 2018-07-21T17:19:51+08:00
draft: true
---

文章结构
	
	1. 实例
	2. 引经据典
	3. 总结

### 实例
首先引入一个代码优化的实例，请参读以下代码：

{% highlight objc %}
- (void)sdkAuthSetServerUrlBase:(NSString *)url isAuthTest:(NSNumber *)test {
	if (test.boolValue) {
		_isSDKAuthTest = test.boolValue;
		if (url && url.length > 0) {
			SdkAuth_setServerUrlBase([url cStringUsingEncoding:NSUTF8StringEncoding]);
		}
	} else {
		if (!_isSDKAuthTest) {
			if (url && url.length > 0) {
				SdkAuth_setServerUrlBase([url cStringUsingEncoding:NSUTF8StringEncoding]);
			}
		}
		_isSDKAuthTest = test.boolValue;
	}
}
{% endhighlight %}
代码格式整齐，变量和函数名命名规范，可以从第一眼看下去，能难理清楚其中的逻辑。该函数是一个供内部测试使用的接口，有两个标准位进行控制。

首先可以把函数的大致逻辑理清楚一下，其主要功能应该是设置sdkAuth的url，于是先把设置url的功能单独拿出来一下，这是在url存在而且有效的时候进行设置，于是函数可以调整为如下：

{% highlight objc %}
- (void)sdkAuthSetServerUrlBase:(NSString *)url isAuthTest:(NSNumber *)test {
	if (test.boolValue) {
		[self setServerUrlBase:url];
	} else {
		if (!_isSDKAuthTest) {
			[self setServerUrlBase:url];
		}
	}
	_isSDKAuthTest = test.boolValue;
}

- (void)setServerUrlBase:(NSString *)url {
	if (url && url.length > 0) {
		SdkAuth_setServerUrlBase([url cStringUsingEncoding:NSUTF8StringEncoding]);
	}
}

{% endhighlight %}
再审视我们的代码，视乎关系变得清楚了一些，但其控制逻辑可以再优化一下，我从满足什么条件的情况下设置url这个角度进行分析。

| test.boolValue | _isSDKAuthTest | setServerUrlBase |
| :------:| :------: | :------: |
| 1 | 0 | true |
| 1 | 1 | true |
| 0 | 0 | true |
| 0 | 1 | false |

从组合情况来看，只有当 test.boolValue 为 false ，_isSDKAuthTest 为 true 的情况下，会不调用 setServerUrlBase 接口，即：

{% highlight objc %}
if(!test.boolValue && _isSDKAuthTest)
{% endhighlight %}

的情况下，不设置。
最终调整版本为：

{% highlight objc %}
- (void)sdkAuthSetServerUrlBase:(NSString *)url isAuthTest:(NSNumber *)test {
	BOOL obsoleteStatus = (!test.boolValue && _isSDKAuthTest);
	if (!obsoleteStatus) {
		[self setServerUrlBase:url];
	}
	_isSDKAuthTest = test.boolValue;
}

- (void)setServerUrlBase:(NSString *)url {
	if (url && url.length > 0) {
		SdkAuth_setServerUrlBase([url cStringUsingEncoding:NSUTF8StringEncoding]);
	}
}
{% endhighlight %}

### 引经据典

以下编程建议，为总结一本经典书籍上的知识点，文章末尾揭示这本书名

1. 要编写整洁的代码，必须先写肮脏代码，然后再清理它。不会一开始写出来的代码就会整洁规范，逻辑清晰的，需要经过不断的修改，优化。先让代码能工作，然后再重构。
2. 今天写的代码，可能在下一版本中被修改，但可读性对以后将要发生的修改有着深远的影响，格式对代码质量很重要；

	垂直格式：
	
		1. 学习报纸的排版，相关性较大的代码放一起
		2. 若某个函数调用了另外一个，就应该把它们放在一起，而且调用者应该尽可能放在被调用者上面
	
	水平格式：
	
		1. 尽力保持代码行简短
		2. 使用空格字符将彼此紧密相关的事物连接到一起，也用空格把相关性较弱的事物隔开
		3. 代码对齐不是硬性要求，但要保证每行代码的长度
	
3. 函数只专注于一件事情，保持代码量少，参数尽量少，没有返回值更好，有返回值的话不要反回空。
4. 函数命名要富有表达性，使被人使用的时候更好理解其作用，多使用专业相关的词汇。
5. 注释方面，要注意不要写得太啰嗦，同时要把自己的意思表达清楚，在需要添加的地方添加
6. 多线程的调用的接口，应该尽量使代码量少，在使用多线程相关的API时要注意：
	 
		1. 使用类库提供的线程安全的群集
		2. 尽可能使用非锁定解决方案
		3. 要注意有哪些类是非线程安全的

### 总结

一方面，需要养成一个在编程中不断学习的习惯，多总结一些好的编程习惯，在平时review代码的时候去实践，发现有不合理的地方，立马修改。有时候可能是一个函数命名或者参数的命名不好，或者在哪里多了一行空格，注释太老了没有更新。

另一方面，多去看看其他人写的代码，学习他人的有点，有不合理的地方提出自己的观点，相互学习共同进步。

程序员代码质量的提高，是一个循序渐进的过程，并不是参考一本书上的编程样式就能马上适应。要在平时的编程中多去实践，提升自己的水平。

参考书籍：《Clean Code》
