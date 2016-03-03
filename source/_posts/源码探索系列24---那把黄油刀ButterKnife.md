 
title: 源码探索系列24---那把黄油刀ButterKnife
date: 2016-03-03 17:45
tags: [android,源码,ButterKnife]
categories: android

------------------------------------------


![enter image description here](https://github.com/JakeWharton/butterknife/raw/master/website/static/logo.png)

相信做安卓开发的人，一定对写一堆 findViewById（）很有印象。特别是当界面多的时候，那简直是觉得为何没有什么简单易行的办法来拯救我们与危难之间呢？

既然我诚心诚意的问了，那就得写下答案，如何解决这个重复累赘问题.


# 解决方案 


<!--more-->
 
## 1.  **使用Live Templates**
  android stuido有一堆的快捷键，其中一个就是解决这个问题的---FBC，这样只要我们在输入的时候按下Alt+Shift+Ctrl+J这组合键。就有下面的内容
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160303105102.png)

接着我们选中那个fbc的，就出现下面的内容。这对于原始手动写还是有一定效率提高。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160303105045.png)

  ![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160303104555.png)

##  2. **用插件**
显然上面这样的确实不错，不过还是得一个一个敲。能否直接一键生成呢？
有的，[Android Layout ID Converter](https://github.com/funnything/OffingHarbor)这个插件可以帮我们快速一键生成。让我们来看下效果。

当我们含辛茹苦的写好我们的xml了之后，只需点Alt+A（我自己修改的）快捷键。就可以快速的生成我们的代码啦，我们选择OK。
![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160303110636.png)

然后再我们的Activity里面粘贴下，就是下面这些内容，是不是有种觉得找到了曙光的感觉？

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160303110704.png)


---
# Butterknife登场

看了这么多，似乎有点跑题？
非也非也。前面铺垫是为了介绍我们的Butterknife实质做的事情而已。

以上，我们最终是希望能够省去这堆find操作，一键生成的，这个就是Butterknife在做的事情和解决的问题。
利用ButterKnife和[Zelezny](https://github.com/avast/android-butterknife-zelezny)插件，我们就可以快速的做到这个效果。先看下插件的示范效果

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_zelezny_animated.gif)

这个插件和前面提到的`OffingHarbor`是同个效果，快速帮我们生成代码。
但也只是生成代码，我们还是需要加多对ButterKnife的依赖，才可以达到和原生的`FindViewById`一样的效果。至于怎么使用这黄油刀，[看下官方答案吧](https://github.com/JakeWharton/butterknife)。就不重复累赘了。

## 原理

我们加入依赖的时候，加入的是下面两个

	compile project(':butterknife')
    apt project(':butterknife-compiler')

嗯，重点在第二个依赖，这个apt（annotation processing tool）project---`butterKnife-compiler`，它的作用就是在『编译』期间动态生成代码（生成的代码由apt负责再编译为class），因此对运行时效率的影响很小。 可以简单的理解为：这个程序可以帮我们`自动的生成FindViewById`代码！

### 基本流程

这个项目会在编译的时候帮我们生成一个`辅助类`(名字格式是 `原类名+$$ViewBinder` )，里面就是我们想偷懒写的代码。

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_QQ%E6%88%AA%E5%9B%BE20160303115559.png)

这里面就是我们想偷懒不写的各种find和setOnClickListener等函数。

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_QQ%E6%88%AA%E5%9B%BE20160303115633.png)

然后我们调用那句	`ButterKnife.bind(this);`的时候，他根据我们传入的这个Activity参数，利用反射找到他生成的那个类，调用其`bind()`方法，执行那堆我们想偷懒不写的代码！（这也很好理解为何是在setContentView后面，因为和我们平时的fbc是一样的）

所以，他相对于我们原生的fbc，主要性能差异在这个反射这个类！但只要放射过一次，他就会缓存下来！因此对性能的影响几乎可以忽略（据闻测试结果在性能差异上小于 20 us，不到1ms啊！！！）

---
### 代码解析
看了原理，当然是开始探索他的源码咯！ 

就先让我们从`ButterKnife.bind(this)` 这句的时候，再说他怎么生成这辅助类的。


	/**
     * Bind annotated fields and methods in the specified {@link Activity}. The current content
     * view is used as the view root.
     *
     * @param target Target activity for view binding.
     */
    public static void bind(@NonNull Activity target) {
        bind(target, target, Finder.ACTIVITY);
    }

	static void bind(@NonNull Object target, @NonNull Object source, @NonNull Finder finder) {
		
	        Class<?> targetClass = target.getClass();
	        ...      
            ViewBinder<Object> viewBinder = findViewBinderForClass(targetClass);
            viewBinder.bind(finder, target, source);   //<--这就是调用辅助类的bind函数，
														//去执行我们不想写的代码的地方
			...														 
    }

然后来看下我们的findViewBinderForClass

	static final Map<Class<?>, ViewBinder<Object>> BINDERS = new LinkedHashMap<>();
	
	@NonNull
    private static ViewBinder<Object> findViewBinderForClass(Class<?> cls)
            throws IllegalAccessException, InstantiationException {
            
        ViewBinder<Object> viewBinder = BINDERS.get(cls);//查缓存，所以下次使用就快啦！
        if (viewBinder != null) {
            if (debug) Log.d(TAG, "HIT: Cached in view binder map.");
            return viewBinder;
        }
        
        String clsName = cls.getName();
        if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
            if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
            return NOP_VIEW_BINDER;
        }
        
        try {
            Class<?> viewBindingClass = Class.forName(clsName + "$$ViewBinder");
           //反射的地方，命名格式：  clsName + "$$ViewBinder"   
                      
            //noinspection unchecked
            viewBinder = (ViewBinder<Object>) viewBindingClass.newInstance();
            
            if (debug) Log.d(TAG, "HIT: Loaded view binder class.");
        } catch (ClassNotFoundException e) {
            if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
            viewBinder = findViewBinderForClass(cls.getSuperclass());
        }
        BINDERS.put(cls, viewBinder);//缓存下来
        
        return viewBinder;
    }

接下来我们去看下那个 viewBinder.bind()的内容，他实现了ViewBinder<T>接口。

	public class SimpleActivity$$ViewBinder<T extends SimpleActivity> implements ViewBinder<T> {
	
	    @Override
	    public void bind(final Finder finder, final T target, Object source) {
	    
	        Unbinder unbinder = new Unbinder(target);
	        View view;
	        
		    ...    
		    
	        view = finder.findRequiredView(source, 2130968578, "field 'hello', method 'sayHello', and method 'sayGetOffMe'");
	        target.hello = finder.castView(view, 2130968578, "field 'hello'");
	        
	        unbinder.view2130968578 = view;
	        view.setOnClickListener(new DebouncingOnClickListener() {
	            @Override
	            public void doClick(View p0) {
	                target.sayHello();
	            }
	        });
	        view.setOnLongClickListener(new View.OnLongClickListener() {
	            @Override
	            public boolean onLongClick(View p0) {
	                return target.sayGetOffMe();
	            }
	        });
	        
	        ...
    }
    
		//解绑，一次性清掉所有的
	    private static final class Unbinder implements ButterKnife.Unbinder {
	        private SimpleActivity target;
	
	        View view2130968578;
	
	        View view2130968579;
	
	        Unbinder(SimpleActivity target) {
	            this.target = target;
	        }
	
	        @Override
	        public void unbind() {
	            if (target == null) throw new IllegalStateException("Bindings already cleared.");
	            target.title = null;
	            target.subtitle = null;
	            view2130968578.setOnClickListener(null);
	            view2130968578.setOnLongClickListener(null);
	            target.hello = null;
	            ((AdapterView<?>) view2130968579).setOnItemClickListener(null);
	            target.listOfThings = null;
	            target.footer = null;
	            target.headerViews = null;
	            target.unbinder = null;
	            target = null;
	        }
	    }
	}
 
写到这里我们看到，他的基本逻辑是这样的

	view = finder.findRequiredView(source, 2130968578, "field 'hello', method 'sayHello', and method 'sayGetOffMe'");
	
	target.hello = finder.castView(view, 2130968578, "field 'hello'");

根据那些view的id，找view，然后cast到特定的类型，赋值给对应绑定的具体的View。(这里的target就是前面传过来的Activity)，所以我们的声明的那些都必须是proteced的，如果是private就不行咯，所以我们可以看到下面的这句错误：

	Error:(37, 22) 错误: @Bind fields must not be private or static. (com.example.butterknife.SimpleActivity.title)

再回过头来看官方的demo，确实是这样，每个前面都不是private的！

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_%E5%82%B2%E6%B8%B8%E6%88%AA%E5%9B%BE20160303131331.png)


好了，接下来我们去看看这个finder里面是什么内容！

	public <T> T findRequiredView(Object source, int id, String who) {
        T view = findOptionalView(source, id, who);
        if (view == null) {
            String name = getResourceEntryName(source, id);
            throw new IllegalStateException("Required view '"+ name+ "' with ID "+ id + 
            " for "+ who+ " was not found. If this view is optional add '@Nullable' 
            (fields) or '@Optional'"+ " (methods) annotation.");
        }
        return view;
    }

	public <T> T findOptionalView(Object source, int id, String who) {
        View view = findView(source, id);
        return castView(view, id, who);
    }
	    
	protected abstract View findView(Object source, int id);
	
	public <T> T castView(View view, int id, String who) {
        try {
            return (T) view;
        } catch (ClassCastException e) {
            if (who == null) {
                throw new AssertionError();
            }
            String name = getResourceEntryName(view, id);
            throw new IllegalStateException("View '"+ name + "' with ID "+ id+ " for "+ 
	            who+ " was of the wrong type. See cause for more info.", e);
        }
    }
    
   -.-    等下，我们好像发现什么不对的， 那个findView居然是个抽象的，背后是谁在干活？
原来我们的Finder是一个枚举的，在我们分析的一开始是这样的，我们传过去的是`Finder.ACTIVITY`
	
	public static void bind(@NonNull Activity target) {
        bind(target, target, Finder.ACTIVITY);
    }
    
再看下源码，这写法，确实没写过！赞

	public enum Finder {
	
	    VIEW {
	        @Override
	        protected View findView(Object source, int id) {
	            return ((View) source).findViewById(id);
	        }
	
	        @Override
	        public Context getContext(Object source) {
	            return ((View) source).getContext();
	        }
	
	        @Override
	        protected String getResourceEntryName(Object source, int id) {
	            final View view = (View) source;	             
	            if (view.isInEditMode()) {
	                return "<unavailable while editing>";
	            }
	            return super.getResourceEntryName(source, id);
	        }
	    },
	    ACTIVITY {
	        @Override
	        protected View findView(Object source, int id) {
	            return ((Activity) source).findViewById(id);
	        }
	
	        @Override
	        public Context getContext(Object source) {
	            return (Activity) source;
	        }
	    },
	    DIALOG {
	        @Override
	        protected View findView(Object source, int id) {
	            return ((Dialog) source).findViewById(id);
	        }
	
	        @Override
	        public Context getContext(Object source) {
	            return ((Dialog) source).getContext();
	        }
	    };
	
		....
		
		protected abstract View findView(Object source, int id);
			
		public abstract Context getContext(Object source);

	}
 
 所以我们看下那个 ACTIVITY 里面的 
 
	        @Override
	        protected View findView(Object source, int id) {
	            return ((Activity) source).findViewById(id);
	        } 

这不就是我们最想偷懒省去的那一行代码吗？
至此，我们的整个过程都看完了，知道这个绑定发生了什么事情了。
那么，现在是时候去看下，他到底是如何生成这个 `xxx$$ViewBinder`类的了

## 编译生成辅助类过程

  ***用运行时annotation预处理技术实现动态的生成代码的***

 

现在我们来看下ButterKnife-compiler里面的butterKnifeProcessor

	@AutoService(Processor.class)
	public final class ButterKnifeProcessor extends AbstractProcessor
	
继承AbstractProcessor（在javax.annotation.processing包里面），关于这个预处理可以简单的理解为，你告诉他你想处理的注解，然后他会返回给你有这些注解的内容给你，之后你自己爱怎么着就怎么着。

### ButterKnifeProcessor

所以我们来看下这个ButterKnifeProcessor的内容，下面截取最重要的部分内容

	@AutoService(Processor.class)
	public final class ButterKnifeProcessor extends AbstractProcessor {
	
		//一堆我们想偷懒的注解方法
		private static final List<Class<? extends Annotation>> LISTENERS = Arrays.asList(//
	            OnCheckedChanged.class, //
	            OnClick.class, //
	            OnEditorAction.class, //
	            OnFocusChange.class, //
	            OnItemClick.class, //
	            OnItemLongClick.class, //
	            OnItemSelected.class, //
	            OnLongClick.class, //
	            OnPageChange.class, //
	            OnTextChanged.class, //
	            OnTouch.class //
	    );
			
		private Elements elementUtils;
	    private Types typeUtils;
	    private Filer filer;
	
	    @Override
	    public synchronized void init(ProcessingEnvironment env) {
	        super.init(env);
	
	        elementUtils = env.getElementUtils();
	        typeUtils = env.getTypeUtils();
	        filer = env.getFiler();
	    }
		
		//通过重载这个方法，返回我们支持的注解类型。
		@Override
		public Set<String> getSupportedAnnotationTypes() {		
		        Set<String> types = new LinkedHashSet<>();
		
		        types.add(Bind.class.getCanonicalName());
		
		        for (Class<? extends Annotation> listener : LISTENERS) {
		            types.add(listener.getCanonicalName());
		        }
		
		        types.add(BindArray.class.getCanonicalName());
		        types.add(BindBitmap.class.getCanonicalName());
		        types.add(BindBool.class.getCanonicalName());
		        types.add(BindColor.class.getCanonicalName());
		        types.add(BindDimen.class.getCanonicalName());
		        types.add(BindDrawable.class.getCanonicalName());
		        types.add(BindInt.class.getCanonicalName());
		        types.add(BindString.class.getCanonicalName());
		        types.add(Unbinder.class.getCanonicalName());
		
		        return types;
		}
		
	  //回调包含注解的内容给我们。让我们自己处理process	
	 @Override
	 public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
	 
	       Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);		
	       
	       for (Map.Entry<TypeElement,BindingClass> entry : targetClassMap.entrySet()) {
	       
	           TypeElement typeElement = entry.getKey();
	           BindingClass bindingClass = entry.getValue();
	           bindingClass.brewJava().writeTo(filer);
		        ...  
	       }		
	      return true;
      
	 }

		@Override
	    public SourceVersion getSupportedSourceVersion() {
	        return SourceVersion.latestSupported();
	    }
			...
	
	}

补充：不想重载也可以这么写

	@SupportedSourceVersion(SourceVersion.RELEASE_7)
	@SupportedAnnotationTypes({"yourAnnotationName"})
	public final class ButterKnifeProcessor extends AbstractProcessor {}

####  findAndParseTargets()
我们来看下这个函数做的什么内容

	 private Map<TypeElement, BindingClass> findAndParseTargets(RoundEnvironment env) {
        Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
        Set<String> erasedTargetNames = new LinkedHashSet<>();

        // Process each @Bind element.
        for (Element element : env.getElementsAnnotatedWith(Bind.class)) {
            if (!SuperficialValidation.validateElement(element)) continue;
            try {
                parseBind(element, targetClassMap, erasedTargetNames);
            } catch (Exception e) {
                logParsingError(element, Bind.class, e);
            }
        }

        ....其余注解的解析

        // Process each @Unbinder element.
        for (Element element : env.getElementsAnnotatedWith(Unbinder.class)) {
            if (!SuperficialValidation.validateElement(element)) continue;
            try {
                parseBindUnbinder(element, targetClassMap, erasedTargetNames);
            } catch (Exception e) {
                logParsingError(element, Unbinder.class, e);
            }
        }

        // Try to find a parent binder for each.
        for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
            String parentClassFqcn = findParentFqcn(entry.getKey(), erasedTargetNames);
            if (parentClassFqcn != null) {
                entry.getValue().setParentViewBinder(parentClassFqcn + BINDING_CLASS_SUFFIX);
            }
        }

        return targetClassMap;
    }

我们就挑`parseBind（）`来看吧，其余类似，就不都贴上来了。

	private void parseBind(Element element, Map<TypeElement, BindingClass> 
							targetClassMap,Set<String> erasedTargetNames) {
	
        // Verify common generated code restrictions.
        if (isInaccessibleViaGeneratedCode(Bind.class, "fields", element)
                || isBindingInWrongPackage(Bind.class, element)) {
            return;
        }

        TypeMirror elementType = element.asType();
        if (elementType.getKind() == TypeKind.ARRAY) {
            parseBindMany(element, targetClassMap, erasedTargetNames);
        } else if (LIST_TYPE.equals(doubleErasure(elementType))) {
            parseBindMany(element, targetClassMap, erasedTargetNames);
        } else if (isSubtypeOfType(elementType, ITERABLE_TYPE)) {
            error(element, "@%s must be a List or array. (%s.%s)", Bind.class.getSimpleName(),
                    ((TypeElement) element.getEnclosingElement()).getQualifiedName(),
                    element.getSimpleName());
        } else {
            parseBindOne(element, targetClassMap, erasedTargetNames);
        }
    }
开头会先检测下是否符合要求 

	  private boolean isInaccessibleViaGeneratedCode(Class<? extends Annotation> annotationClass,
                                                   String targetThing, Element element) {
        boolean hasError = false;
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

        // Verify method modifiers.
        Set<Modifier> modifiers = element.getModifiers();
        
        //这个限定条件我们在开头有说过，在这里看到了！对于private的他在Bind()函数里面是没法直接赋值
        //所以他对这些访问修饰符要做下判断
        if (modifiers.contains(PRIVATE) || modifiers.contains(STATIC)) {
            error(element, "@%s %s must not be private or static. (%s.%s)",
                    annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                    element.getSimpleName());
            hasError = true;
        }

        // Verify containing type.
        if (enclosingElement.getKind() != CLASS) {
            error(enclosingElement, "@%s %s may only be contained in classes. (%s.%s)",
                    annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                    element.getSimpleName());
            hasError = true;
        }

        // Verify containing class visibility is not private.
        if (enclosingElement.getModifiers().contains(PRIVATE)) {
            error(enclosingElement, "@%s %s may not be contained in private classes. (%s.%s)",
                    annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                    element.getSimpleName());
            hasError = true;
        }

        return hasError;
    }

另外看下那个对包的检测

	   private boolean isBindingInWrongPackage(Class<? extends Annotation> annotationClass,
                                            Element element) {
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
        String qualifiedName = enclosingElement.getQualifiedName().toString();

        if (qualifiedName.startsWith("android.")) {
            error(element, "@%s-annotated class incorrectly in Android framework package. (%s)",
                    annotationClass.getSimpleName(), qualifiedName);
            return true;
        }
        if (qualifiedName.startsWith("java.")) {
            error(element, "@%s-annotated class incorrectly in Java framework package. (%s)",
                    annotationClass.getSimpleName(), qualifiedName);
            return true;
        }

        return false;
    }
    
对于系统框架包的内容他是不支持的。所以在我们的程序最好是不要加这两个关键字眼哈。
我们继续看回原来的剩下内容

		TypeMirror elementType = element.asType();
		
        if (elementType.getKind() == TypeKind.ARRAY) {
            parseBindMany(element, targetClassMap, erasedTargetNames);
        } else if (LIST_TYPE.equals(doubleErasure(elementType))) {
	        //这两个判断条件看起来就奇妙了，为何不合并哈,都是调用同个方法的	        
            parseBindMany(element, targetClassMap, erasedTargetNames);
        } else if (isSubtypeOfType(elementType, ITERABLE_TYPE)) {
            error(element, "@%s must be a List or array. (%s.%s)", Bind.class.getSimpleName(),
                    ((TypeElement) element.getEnclosingElement()).getQualifiedName(),
                    element.getSimpleName());
        } else {
            parseBindOne(element, targetClassMap, erasedTargetNames);
        }

我们挑个简单的parseBindOne来看

	private void parseBindOne(Element element, Map<TypeElement, BindingClass> targetClassMap,
                              Set<String> erasedTargetNames) {
        boolean hasError = false;
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

        // Verify that the target type extends from View.
        TypeMirror elementType = element.asType();
        if (elementType.getKind() == TypeKind.TYPEVAR) {
            TypeVariable typeVariable = (TypeVariable) elementType;
            elementType = typeVariable.getUpperBound();
        }
        
        ...
        
        // Assemble information on the field.
        int[] ids = element.getAnnotation(Bind.class).value();
        
        ...

        int id = ids[0];
        BindingClass bindingClass = targetClassMap.get(enclosingElement);
	    //查重，一个id只给绑定一个view。像下面这种就会报错。为何不给重复呢？
	    //@Bind(R.id.footer)
	    //TextView footer;
	    //@Bind(R.id.footer)
	    //TextView tryErrorView;
    
        if (bindingClass != null) {
            ViewBindings viewBindings = bindingClass.getViewBinding(id);
            if (viewBindings != null) {
                Iterator<FieldViewBinding> iterator = viewBindings.getFieldBindings().iterator();
                if (iterator.hasNext()) {
                    FieldViewBinding existingBinding = iterator.next();
                    error(element, "Attempt to use @%s for an already bound ID %d on 
                    '%s'. (%s.%s)",Bind.class.getSimpleName(), 
                    id,existingBinding.getName(),                            
                    enclosingElement.getQualifiedName(), element.getSimpleName());
                    return;
                }
            }
        } else {
            bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);
        }

        String name = element.getSimpleName().toString();
        TypeName type = TypeName.get(elementType);
        boolean required = isFieldRequired(element);

        FieldViewBinding binding = new FieldViewBinding(name, type, required);
        bindingClass.addField(id, binding);//保存这个bind,后面在生成类文件时候要用到！

        // Add the type-erased version to the valid binding targets set.
        erasedTargetNames.add(enclosingElement.toString());
    }

得看下那个getOrCreateTargetClass()里面的内容是什么

	private BindingClass getOrCreateTargetClass(Map<TypeElement, BindingClass> targetClassMap,
                                                TypeElement enclosingElement) {

        BindingClass bindingClass = targetClassMap.get(enclosingElement);
        if (bindingClass == null) {
            String targetType = enclosingElement.getQualifiedName().toString();
            String classPackage = getPackageName(enclosingElement);
            String className = getClassName(enclosingElement, classPackage) + BINDING_CLASS_SUFFIX;
            //BINDING_CLASS_SUFFIX = "$$ViewBinder",我们提到过的命名格式
            bindingClass = new BindingClass(classPackage, className, targetType);
            targetClassMap.put(enclosingElement, bindingClass); 
        }
        return bindingClass;
    }
    
这个BindingClass会保存我们要生成的辅助类的信息。后面会用到


这样我们假设我们只有@Bind要解析，默认其余的没有（其实就是懒）。
这样我们的findAndParseTargets()就搞定了。继续会主线

	    @Override
    public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
        Map<TypeElement, BindingClass> targetClassMap = findAndParseTargets(env);

        for (Map.Entry<TypeElement, BindingClass> entry : targetClassMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingClass bindingClass = entry.getValue();
             
            bindingClass.brewJava().writeTo(filer);  //  <--- 我们的关注点
			...
        }
        return true;
    }

让我们来看下那个brewJava（）

	JavaFile brewJava() {
        TypeSpec.Builder result = TypeSpec.classBuilder(className)
                .addModifiers(PUBLIC)
                .addTypeVariable(TypeVariableName.get("T", ClassName.bestGuess(targetClass))); 

        if (parentViewBinder != null) {
            result.superclass(ParameterizedTypeName.get(ClassName.bestGuess(parentViewBinder),
                    TypeVariableName.get("T")));
        } else {
            result.addSuperinterface(ParameterizedTypeName.get(VIEW_BINDER, TypeVariableName.get("T")));
        }        
        
	//上面几行负责生成源文件的类头,如果没有parentViewBinder的话类似下面这样的
	//public class SimpleActivity$$ViewBinder<T extends SimpleActivity> implements ViewBinder<T> 

        if (hasUnbinder()) {
            result.addType(createUnbinderClass());//我们的unBider内容
        }

        result.addMethod(createBindMethod()); // 生成我们的一堆bind内容

        return JavaFile.builder(classPackage, result.build())
                .addFileComment("Generated code from Butter Knife. Do not modify!")
                .build();
    }
    
通过这样的配置，我们的生成文件结构类似这样

![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/dagger_%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20160303165317.png)

我们挑着createBinderMethod来看

	private MethodSpec createBindMethod() {

			
        MethodSpec.Builder result = MethodSpec.methodBuilder("bind")
                .addAnnotation(Override.class)
                .addModifiers(PUBLIC)
                .addParameter(FINDER, "finder", FINAL)
                .addParameter(TypeVariableName.get("T"), "target", FINAL)
                .addParameter(Object.class, "source");
//配置我们的函数名，如图片所示的那个bind()方法 
 
       ...
      // 下面是我们比较关心的一个点
        if (!viewIdMap.isEmpty() || !collectionBindings.isEmpty()) {
            // Local variable in which all views will be temporarily stored.
            result.addStatement("$T view", VIEW);

            // Loop over each view bindings and emit it.
            for (ViewBindings bindings : viewIdMap.values()) {
                addViewBindings(result, bindings);
            }

            // Loop over each collection binding and emit it.
            for (Map.Entry<FieldCollectionViewBinding, int[]> entry : collectionBindings.entrySet()) {
                emitCollectionBinding(result, entry.getKey(), entry.getValue());
            }
        }

       ...

        return result.build();
    }
这个`viewIdMap`保存着我们前面在`parseBindOne`里面调用的`bindingClass.addField(id, binding);`保存下来的信息。

	private void addViewBindings(MethodSpec.Builder result, ViewBindings bindings) {
        List<ViewBinding> requiredViewBindings = bindings.getRequiredBindings();
        if (requiredViewBindings.isEmpty()) {
            result.addStatement("view = finder.findOptionalView(source, $L, null)", bindings.getId());
        } else {
            if (bindings.getId() == NO_ID) {
                result.addStatement("view = target");
            } else {
                result.addStatement("view = finder.findRequiredView(source, $L, $S)", bindings.getId(),
                        asHumanDescription(requiredViewBindings));
            }
        }

        addFieldBindings(result, bindings);//生成绑定方法
        addMethodBindings(result, bindings);//生成监听方法
    }

这个方法生产我们前面在说辅助类里面提到的两句话和一堆的监听的方法！ 

经过这一轮的配置，我们看到了曙光！终于生成了一个JavaFile对象了！！
可以开始写内容了


	bindingClass.brewJava().writeTo(filer); 

根据返回的JavaFile对象，调用其writeTo()方法 

但是呢，这个javaFile是javapoet包的代码，而[javapoet](https://github.com/square/javapoet)**是一个生成java源文件的框架**。而且也是他们Square的，嗯，我就不继续翻下去了，至此在编译期的处理结束。
（java诗人，这名字好，啊哈哈） 
    

# 小结

我们看到整个做了这么多的工作，就是为了我们偷懒那么几行代码，真是**台上一秒钟，台下十年工**的即视感。

但这整个框架真的给我们的开发带来便利了吗？个人觉得是需要权衡下的。
如果只是偷懒不想写fbc。那么用前面提到的插件完全可以解决。
至于onClick，我们可以写在xml里面

	  <Button
        android:id="@+id/hello"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_margin="10dp"
        android:onClick="onHelloClick"
        android:layout_weight="1"
        />
        
不过它还是提供了不少这个别的监听事件。如果没有什么要求，我还是倾向于用古董的方法。

#  后记

类似的还有`Dagger2`，我们看到市面上的注入还有xUtil这种，但他是每次都反射，效率和前者的不是一个等级上的。 写到现在，好像常见的第三方框架不少是Square开源的。不得不说这家公司还真的挺正的。
Retrofoi + okHttp + leakcanary +okio  +picasso +otto + dagger等等！


我的翻译工作...一个礼拜过去，才翻译了第一章的一节，就几千个字。
我这种新手还是太`TOO YOUNG,TOO SIMPLE`，居然夸大海口。哈哈哈！！！
我就是跪着也不太可能三个礼拜翻译完了！