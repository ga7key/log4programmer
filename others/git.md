### git commit message form

commit message一般包括3部分：Header、Body、Footer。

```
<type>(<scope>):<subject>
blank line
<body>
blank line
<footer>
```

**header**是必需的，body、footer可以省略。  
header中**type**、**subject**是必需的，scope可以省略。  

---

##### type  

用来说明commit的类别：

 - feat：新功能  
 - fix：bug修复  
 - docs：文档  
 - style：格式（与功能无关）  
 - refactor：重构  
 - perf：优化
 - test：测试相关  
 - chore：构建过程或辅助工具的变动  

type是feature和fix时，则该commit会自动加载到 change log 中。其它type类型可选配，建议不要加入到 change log。  

##### scope  

用于说明commit影响的范围，比如数据层、控制层、视图层等。  
如果修改不止影响一个scope，可以用`*`表示。  

##### subject  

对当前commit内容的简要描述。  

##### Body  

对当前commit内容的详细描述。  

##### Footer  

只用于以下两种情况：  

- 不兼容变动：如果当前版本与上一个版本不兼容，则Footer部分以BREAKING CHANGE开始，加上详细内容。
- 关闭Issue：如果当前commit针对某个issue，则在footer部分关闭这个issue。  

##### Revert  

特殊情况，当前的commit用于撤销之前的commit时，必须以revert开头，加上将要被撤销的commit的Header。  
Body格式固定，`This reverts commit `加上将被撤销的commit的SHA标识符。