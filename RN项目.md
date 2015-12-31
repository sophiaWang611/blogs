### 性能篇

总体上来说，RN项目在production模式下性能还是可以的，页面滑动和切换都很丝滑。
但是在开发模式下，复杂页面的卡顿很是明显，比如订单列表页、菜品列表页。

显而易见，包含列表的页面在卡顿问题上很突出。那我们就来回忆一下列表的优化过程。

#### 分批渲染
首先，我们是先引入了“setTimeout”，让获取数据的方法延后500ms执行，可是延迟的弊端在切换到production模式后就立马就显现出来了。
在生产环境下，页面渲染速度有所优化，致使这延迟的500ms反倒成了页面渲染速度慢的原因。

查看FB的文档后，我们发现了“InteractionManager.runAfterInteractions”，这个方法会在页面交互完成之后被调用，正是我们所需要的！
于是我们将复杂度比较高的页面放到这个回调中绘制，以达到分层渲染的效果。代码如下：

在componentDidMount时，获取页面数据（这部分放在didmount和willmount差别都不大）。

	componentDidMount: function () {
		this.getOrderList(1);
	},

获取到数据后不马上触发页面渲染，而是等页面交互完成之后。
页面首先渲染导航栏以及页面切换时的动画，然后绘制订单列表。

	getOrderList: function(currentPage,) {
		OrderService.getOrderList(currentPage, 
			(resp)=>{
				.... ....
				InteractionManager.runAfterInteractions(()=>{
					this.setState({
						dataSource: this.state.dataSource.cloneWithRows(dataList),
						... ...
					});
				});
		});
    }

至此，切换到订单列表页时的卡顿消失了。

#### 拆分业务组件，并缓存已创建的ReactElement
首次渲染页面卡顿的问题解决后，我们又可以愉快得在光滑的页面上面摩擦摩擦。
才觉得幸福到来的时候，产品🐶拎着倒计时功能和格式各样的按钮出现了。

原本列表只显示订单的基本信息，每次只在翻页时渲染新页的5个item，渲染成本并不高。
但是一旦引入倒计时功能，就意味着每个row每一秒都需要重绘。由于row和列表写在一个组件中，随之而来就是列表的每秒重绘。
不敢想翻到后几页时，渲染进程会忙成什么样纸....

想到此，我们决定重构。先把每行提成独立的组件，再把每行的内容根据功能块切分成多个业务组件：状态栏（倒计时所在组件），按钮栏，菜单栏...
随着业务组建的提取，页面重绘的scope随之变小。原本影响着整个列表重绘的倒计时功能，现只影响状态栏组件。
代码目录:
	
	OrderList.js
		render() {
			return(
				<MSRefreshableList  renderRow={()=>{return (<OrderRow order={orderItem})}} ... .../>
			);
		}

	OrderRow.js
		render() {
			return (
				<OrderStatus>...</OrderStatus>
				.... ....
				<OrderButton>...</OrderButton>
			)
		}
		

解决了倒计时，下面我们来说按钮。
绘制按钮并不会增加渲染的负担，但是按钮所绑定的操作事件会改变订单的状态，影响到订单的排序以及出现哪一个列表。

这一次引起的重绘不比之前，是不可避免的了。我们唯一能做的就是尽可能的减少不必要的绘制。
所幸的是大部分的订单信息并无改变，我们所要重绘的不过是值改变了的row。

鉴于此，我们尝试将每次返回的OrderRow以orderId为key缓存起来。
renderRow调用时先比较Order对象有无变化，若变化了则新建，并存入缓存；若没有则取缓存找，找不到才新建。
代码如下：
	
	renderRow(orderItem) {
		var cachedOrder = this.orders[orderItem.orderId];
		if (cached && _.isEqual(cached, orderItem)) {
			return this.renderedRows[orderItem.orderId];
		}
		var rendered = (<OrderRow order={orderItem}/>);
		this.orders[orderItem.orderId] = orderItem;
		this.renderedRows[orderItem.orderId] = rendered;
		return rendered;
	}

运行之后，神奇的发现性能有很大的提升。同时还解决了从子页面退回到列表页时页面卡顿的问题（为何后退会让页面卡顿，会在后面详细说明）。


#### 正确理智地设值
发现缓存可以大幅提高列表渲染速度后，小伙伴们纷纷在自己写的页面或组件里面尝试。但是有一个组件无论如何缓存都没有任何作用，还是该卡卡该慢慢。
几轮调试后，终于，又一个大坑浮出水面。
(未完待续...)



#### ListView  
	pageSize等属性

#### Navigator
	willforce等属性

#### “shouldComponentUpdate”方法
	减少不必要的渲染
