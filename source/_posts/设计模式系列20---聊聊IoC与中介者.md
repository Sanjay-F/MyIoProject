title: 设计模式系列20---聊聊IoC与中介者
date: 2016-01-05 15:46 
tags: [android,设计模式,pattern,Memento]
categories: android

------

有一个叫`控制反转`(Inversion of Control，缩写IoC ) 的东西 ，这个对于计算机的人应该是不陌生的概念，就算你不知道那个`Bob大叔`。

这个概念简单说的是下面这样的事情

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/%E4%B9%B1%E9%BA%BB_%E8%80%A6%E5%90%88%E5%85%B3%E7%B3%BB.JPG)

原本各个类之间的关系乱七八糟的，看起来头都晕了。
他们就像齿轮一样，相互咬合依赖。

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/%E9%BD%BF%E8%BD%AE_%E8%80%A6%E5%90%88%E5%85%B3%E7%B3%BB_full.jpg)
如果有一个出问题，那可能整个就崩溃了。

但是，如果有一个人出来承担大事，负责协调各个类的话！
那么，他们的关系就可以是像下面图片这样的，
各个对象之间可以不感受到对方的“存在”，但整体又运作良好。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/%E9%BD%BF%E8%BD%AE_%E8%A7%A3%E8%80%A6%E5%90%88_full.jpg)

 举个可能不恰当的例子：
 
 企业`ObjectA`要招人，可能要去学校`ObjectB`开宣讲会，去别的企业`ObjectC`挖人等等。
 有些人看到了其中的机会，专门负责为企业输送人才，我们称这类人叫猎头`Ioc`（也有人才中心等，这里为描述方便，不细究），他负责帮助企业去找到需要的人，我们的HR就可以在公司呆着，看着猎头源源不断发来简历就好了！
 再也不用去和学校，别的企业打交道了。是的，从此HR与这两者解耦合,过上了幸福美好的生活!!

那么问题来了，如何用中介者模式来模拟这个招聘流程？
 
今天我们要聊的这个就中介者就是类似的情况，我们来看下实际的代码情况。
 

<!--more-->

# 代码实现

现在来看下我们的公司吧

	public class Company extends AbstractColleague {
	
	    public Company(AbstractMediator abstractMediator) {
	        super(abstractMediator);
	    }
	
	    public void seeResume(String resume) {
	        System.out.println("正在看简历： " + resume);
	    }
	
	    public void startToHunt() {
	        abstractMediator.execute(AbstractMediator.Action.START_HIRE);
	    }
	}
 我们的公司主要工作就是看简历，发通知说我要开始招人了，为了方便说明，就把后续的面试环节剩了。

另外，我们要求实现`AbstractColleague`,用于对外沟通的地方，是我们的齿轮与IOC发生联系的基础。

	public abstract class AbstractColleague {
	
	    AbstractMediator abstractMediator;
	
	    public AbstractColleague(AbstractMediator abstractMediator) {
	        this.abstractMediator = abstractMediator;
	    }
	}
 
  然后我们的学生也类似
  
	public class Student extends AbstractColleague {
		
	    public Student(AbstractMediator abstractMediator) {
	        super(abstractMediator);
	    }
	
	    public String getMyResume() {
	        return "我的简历很简单";
	    }
	
	    public void findJob() {
	        abstractMediator.execute(AbstractMediator.Action.FIND_JOB_SCHOOL, getMyResume());
	    }
	}
找工作主要通过校招的途径。

而对于这类的已经在上班的人啊，主要就是社招来跳槽了。


	public class Employee extends AbstractColleague {
	
	    public Employee(AbstractMediator abstractMediator) {
	        super(abstractMediator);
	    }
	
	    public String getMyResume() {
	        return "这是我的简历,项目经验很多";
	    }
	
	    public void findJob() {
	        abstractMediator.execute(AbstractMediator.Action.FIND_JOB_SOCAIL, getMyResume());
	    }
	
	    public void refuseOffer() {
	        System.out.println("这公司我不去");
	    }
	
	}


好了，准备好这么多，得来看下我们的`猎头`了

	public abstract class AbstractMediator {
	
	    protected Company mCompany;
	    protected Student mStudent;
	    protected Employee mEmployee;
	
	    public AbstractMediator() {
	
	        mCompany = new Company(this);
	        mStudent = new Student(this);
	        mEmployee = new Employee(this);
	    }
	
	    public abstract void execute(Action action, Object... objects);
	
	    public enum Action {
	        FIND_JOB_SCHOOL, FIND_JOB_SOCAIL,START_HIRE
	    }
	
	}

我们抽象出猎头和学生，公司打交道，所以有这么几个内部变量，另外他的基本行为就是帮人找工作，执行特定的行为`execute()`，但具体怎么做，就见仁见智了。

例如下面这位，他对好简历的要求是最少有10个字。少于十个字的都不帮你交出去，免得丢了自己的招牌。

	public class HeadMediator extends AbstractMediator {
	
	    private int GOOD_RESUME = 10;	
	    private boolean isCompanyStartToHire = false;
	
	    @Override
	    public void execute(Action action, Object... objects) {
	        switch (action) {
	            case FIND_JOB_SCHOOL:
	                findJobBySchool((String) objects[0]);
	                break;
	            case FIND_JOB_SOCAIL:
	                findJobBySocial((String) objects[0]);
	                break;
	            case START_HIRE:
	                isCompanyStartToHire = true;
	                break;
	        }
	    }
	
	    public void findJobBySocial(String resume) {
	        if (isCompanyStartToHire && resume.length() > GOOD_RESUME) {
	            super.mCompany.seeResume(resume);
	        }
	    }
	
	    public void findJobBySchool(String resume) {
	        if (isCompanyStartToHire && resume.length() > GOOD_RESUME) {
	            super.mCompany.seeResume(resume);
	        }
	    }
	}

基本我们准备好了，来看下我们的测试案例

	public class Client {
	
	    public static void main(String[] args) {
	
	        AbstractMediator mediator=new HeadMediator();
	        	
	        Company company=new Company(mediator);
	        company.startToHunt();//公司说开始招人人	
 
	        Student student=new Student(mediator);
	        student.findJob();//学生开始找工作
	
	        Employee employee=new Employee(mediator);
	        employee.findJob();//员工想跳槽，开始找工作	
	    }
	}
 
# 类图

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160105152709.png)


# 后记

这个中介者模式到目前位置，在我的开发羡慕中是没有实际的自己手动写过。
不过倒是用过不少类似的库，例如我们的EventBus，OTTO，广播等。

通过这个中介者模式，具体的活到底是怎样的流程，我们全部进行了抽象，交给了中介者，各个类需要做的就是专注于自己本身的行为。当然，这样的高度控制集中化也是有点小小问题的，当涉及到的类多起来，通讯复杂的时候，这个类要写的内容还是不少的，这可能使他难以维护！


#参考资料

[架构师之路(39)---IoC框架](http://blog.sina.com.cn/s/blog_8b7263d10101agyd.html) 
 