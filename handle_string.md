## 字符串处理

* ${#string} 字符串的长度

提取子串

* ${string:position:length} 提取string中从position位置开始的length个字符，如果length省略，则到字符串末尾，`需要注意的是：当要从字符串末尾定位时，可以将position设为负数，但是，要在string和position之间添加空格，或者将position用()包括起来`

子串削除

* ${string#substring} 从string的`开头`位置截掉最`短`匹配的substring
* ${string##substring} 从string的`开头`位置截掉最`长`匹配的substring
* ${string%substring} 从string的`结尾`位置截掉最`短`匹配的substring
* ${string%%substring} 从string的`结尾`位置截掉最`长`匹配的substring

子串替换

* ${string/substring/replacement} 将string中的第一个匹配substring的字符串替换为replacement
* ${string//substring/replacement} 将string中的所有匹配substring的字符串替换为replacement
* ${string/#substring/replacement} 将string的`开头`的substring替换为replacement
* ${string/%substring/replacement} 将string的`结尾`的substring替换为replacement

参数替换

${parameter}
${parameter-default} 如果变量parameter没有被`声明`，那么就使用默认值
${parameter:-default} 如果变量parameter没有被`设置`，那么就使用默认值

注意：
声明和设置是有差别的：声明只是说有这个变量，而设置是要为这个变量赋值，且不能赋空值。因此，上面的差别就是，当`para=`时，变量para被声明，但是没有被设置。

${parameter=default} 如果变量parameter没有被`声明`，那么`parameter=default`
${parameter:=default} 如果变量parameter没有被`设置`，那么`parameter=default`

${parameter+value} 如果变量parameter`被声明`，那么就使用value，否则就使用null
${parameter:+value} 如果变量parameter`被设置`，那么就使用value，否则就使用null

${parameter?err_msg} 如果变量parameter`被声明`，那么就使用声明的值，否则打印err_msg错误消息
${parameter:?err_msg} 如果变量parameter`被设置`，那么就使用设置的值，否则打印err_msg错误消息