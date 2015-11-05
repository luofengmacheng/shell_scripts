## shell脚本学习

### 1 参数获取之shift和getopt

执行shell脚本时，经常需要给脚本传递参数，而脚本就需要处理这些参数，shift可以帮助我们获取参数。

脚本接收到参数之后，可以通过$n获取第n个参数，例如，./test.sh a b c，那么$0就是./test.sh，$1就是a，$2就是b，$3就是c。而且，为了使得参数能够表示不同的含义，通常会采用跟linux一样的输入参数的形式：-type content，type表示参数的含义，content就是参数的值。

方式1：

用一个变量定位到第一个参数，然后，每获取一个参数，就将该变量加1或者2。但是，实测发现，该方式不太可行。一种与之类似的方法是将参数保存到数组中，然后用一个索引遍历该数组。

方式2：

使用shift可以将参数左移，例如，shift 2表示丢弃$1和$2，即$3变成1，$4变成2。

``` shell
while [ ! -z $1]
do
	case $1 in
		-h) h=$2; shift 2;;
		-a) a=$2; shift 2;;
		*) ;;
	esac
done
```

上例可以获取参数中的-h和-a选项，首先查看$1是否为空，如果不空，则判断$1为-h或者-a，然后获取参数的值，再将刚刚的选项和参数值都丢弃，那么$1就是新的选项，$2就是新的参数值。

另一种方式是采用getopts和getopt，采用这种方式，需要了解传递参数的几种方式：

* -a -b -c 短选项，且选项不需要参数
* -abc 短选项，和上面的一样，只是合并了
* -a args -b -c 短选项，只是-a需要参数，-b和-c不需要参数
* --a=args --b 长选项

方式3：getopts

getopts是bash内置的命令，但是，它不支持长选项。

``` shell
while getopts "a:bc" arg
do
	case $arg in
		a) echo $OPTARG;;
		b) echo "b";;
		c) echo "c";;
		?) ;;
	esac
done
```

其中，getops后面的字符串表示选项，后面有:的表示它需要参数，而且参数值存储在环境变量$OPTARG中。

因此，上面获取参数的意思是：循环获取参数，并将获取的参数的选项放到arg中，参数值放到$OPTARG中。

方式4：getopt

getopt是独立的程序，并且支持长选项。

``` shell
// -o表示短选项，一个冒号表示需要选项值，两个冒号表示选项值可选
// 因此，a选项不需要值，b选项需要值，c选项的值可选
TMP=`getopt -o ab:c:: --long a-long,b-long:,c-long:: -- "$@"`

eval set -- "$TMP"

while true; do
	case "$1" in
		-a|--a-long) echo "a"; shitf;;
		-b|--b-long) echo "b"; shift 2;;
		-c|--c-long) echo
		case "$2" in
			"") echo "c"; shift 2;;
			*) echo "参数值为'echo $2'"; shift 2;;
			esac;;
	--) shift; break;;
	*) echo "error"; exit 1;;
	esac
done
```

### 2 点命令