---
layout: post
comments: true
title:  "About MVVC Issue"
excerpt: "About MVVC Issue"
date:   2019-08-07 22:00:00
---

### About MVVC Issue
# 一、关于过滤功能的一点总结 #
## 项目原始代码如下 ##
**每一列局部调用的filter-method方法**

    filterHandler(value, row, column) {
        let _this = this;
        const property = column['property'];
        debugger
        return row[property] === value;
      },

**表格整体全局调用的filter-change方法**

    handleFilterChange(filters){ //filter={"column_name":["type_1","type_2","type_3"]}
                let keys = Object.keys(filters); //keys=["column_name"]
                let newArr = [];
                keys.forEach(key=>{
                    newArr = filters[key] //newArr=["type_1","type_2","type_3"]
                });
                if(newArr.length > 0){ //newArr.length=1
                    debugger
                    this.tableData = this.originData.filter(item => {
                        return newArr.indexOf(item[keys[0]]) !== -1;
                    });
                    debugger
                    this.storage.pageNo = 1;
                }else{
                    this.tableData = this.originData
                }
            },

## 其中存在以下问题 ##
- 当连续点选不同列的筛选框时，会出现总行数（行数）即`tableData.length!=filteredData.length`的情况，呈现到页面上就是显示结果有100条数据，分页正常，到实际表格中显示为0条
## 分析问题根源 ##
> *在业务功能开发过程中遇到问题，首先需要明确的是用户想要什么，而不能自觉地在代码逻辑上做出让步。钻研下去，就可以摘到星星和月亮。*

> **通过跟踪代码复现异常，我们定位到了问题的本质原因。同时发现了原有代码中的冗余，即直接将 `handleFilterChange(filters)` 函数完全删除，对整个系统没有丝毫影响。表格在筛选数据的时候，在每一列上逐行逐列执行`filterHandler`方法就得到了表格要呈现的内容，handleFilterChange完全是做了多余的过滤。这就相当于表格先将数据过滤了一遍，然后又交给每一列再过滤一遍。**

> **进一步深入研究，发现每列筛选后最终呈现的条目数都是正确的，错误的只是分页导航栏目中的总数目是错误的。所以对问题认识的关键点在于，表格呈现是正确的，分页栏目总数是错误的。**

    ## 1.保持表格呈现内容不变；2.修正过滤后的总条目数 ##

> ## 具体解决bug的方法 ##

> **1. 调用框架函数法：**最直接容易想到的方法

> 经过查阅资料，`elementUI`中的过滤筛选内部机制是，自动化地将内部数据备份了一遍，而过滤完之后为提供任何方法来获取过滤后的值。所以调用表格自带方法来获取值的方法不可取。

> **2. 增加计数变量法：**间接可以想到的方法

> 通过在`filterHandler`中加入计数器，来获取过滤后的总数。由于过滤器是逐列执行的，这样就需要每列定义一个计数器。而实际计算当中，发现计数器返回的值不符合实际，猜测可能这个过滤筛选是并发执行的，或者内在有特殊机制，所以不可取。

> **3. 数据分析法**：需要琢磨和灵感的方法

> 通过制作`Excel`表格，分情况讨论后进行数据分析发现，过滤执行在`原始数据`和`当前数据`上两种情况下时，只要`当前数据`执行后的数据总量值为下降的，那么这个下降后的总行数值一定是页面实际展示的正确值。但当其不变时，就无法确定。

> **4. 引入堆栈法：**认真剖析问题总结后的方法

> 引入`栈`固然是个很好的解决方法。但是需要精确掌握`table`中的事件监听机制：即`table`每次只能检测到一个事件的变化，而且对于`加选`、`减选`、`连选`三者的感受情况不一样，以及用户是在已经选过的栏目加选，还是在新的栏目加选，或者在已选栏目中即加选又减选等复杂操作。上述多种情况处理完成后，还需要对用户的重置操作做出正确的处理，比如用户是只想重置一栏，还是全部过滤条件都想重置等。

**显然，将上述第3、4中方法合并起来，细致设计，最终就能解决问题。**

> ## 拟定解决方案 ##
> 由于周末不上班，未能在现场编写代码解决，通过思维导图形式暂时将逻辑理清如下：
修改的主要代码部分：

    handleFilterChange(filters){//传入监听到的过滤器列，输出正确过滤后总行数值totalSize
		/** 
		this.filtersStack=
			{
				"colume_1":["type_1","type_2","tyoe_3","..."],
				"colume_2":["type_1","type_2","tyoe_3","..."],
				"colume_3":["type_1","type_2","tyoe_3","..."],
			}; 
		*/
		//旧数据出栈，新数据入栈
		this.filtersStack.remove(Object.keys(filters));//或者reset为空
		this.filtersStack.push(filters);//或者set为filters[Object.keys(filters)]
		
		if（this.searchKey.length>0）//进入搜索模式
		{	
			this.tableData = this.tableDataBackup;//在原始数据上过滤
			for key in Object.keys(filtersStack)//逐列逐条件过滤
			{
				var currentFilterArray = filtersStack[key];
				if(currentFilterArray.length > 0)//存在过滤条件的情况下
				{
		        	debugger
		        	this.tableData = this.tableData.filter(item => 
						{
		        		return currentFilterArray.indexOf(item[key]) !== -1;
						});
				}else//情空或者重置过滤条件的情况下，数据不变
				{
					this.tableData = this.tableData；
				}
			}
		}else//进入全局模式（非搜索模式）
		{
			this.tableData = this.tableDataSearched;//在搜索到的数据上过滤
			for key in Object.keys(filtersStack)//逐列逐条件过滤
			{
				var currentFilterArray = filtersStack[key];
				if(currentFilterArray.length > 0)//存在过滤条件的情况下
				{
		        	debugger
		        	this.tableData = this.tableData.filter(item => 
						{
		        		return currentFilterArray.indexOf(item[key]) !== -1;
						});
				}else//情空或者重置过滤条件的情况下，数据不变
				{
					this.tableData = this.tableData；
				}
			}
		}
		this.storage.pageNo = 1;//重置页码为第一页
		this.storage.totalSize = this.tableData.length;//这样就能保证过滤后总行数、页码数一致了，并且呈现给用户正确的结果，排除所有异常
	}

## 结语 ##
> 有时候项目需要快速迭代、交付时间紧迫，问题频出应接不暇。只要深入钻研，采取多种方法展开攻坚战，问题常常就能迎刃而解。反之，若知难而退，捂住盖子，终酿成大问题。