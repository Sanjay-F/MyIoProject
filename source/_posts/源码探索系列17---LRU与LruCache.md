title: 源码探索系列17----LRU与LruCache
date: 2016-01-07 19:07
tags: [android,源码,LRU,LruCache]
categories: android
------------------------------------------ 
 


![enter image description here](http://7xl9zd.com1.z0.glb.clouddn.com/a08b87d6277f9e2fe1bb18c01e30e924b999f368.jpg)
前段时间的文章都配图，今天写完才觉得少了什么，想起来要图，所以今天我们就补一只逗币的哈士奇在这里吧。哈哈


今天我们来聊聊那个LRU（Least Recently Used 近期最少使用算法）在我们开发过程的实际应用和我们的安卓系统在support包有提供的一种实现`LruCache`。

在开始讲它之前，我们来看下他的基本使用。

<!--more-->

	class CacheEngineLruCache {
	
	    private static final String TAG = "CacheEngineLruCache";	
	    LruCache<String, Object> mMemoryCache = null;	
	    static final int MAX_CACHE_NUM = 2000;
	
	    protected CacheEngineLruCache() {
	        mMemoryCache = new LruCache<String, Object>(MAX_CACHE_NUM);
	    }
	 
	    public Object get(String key) {
	        return mMemoryCache.get(key);
	    }
	  
	    public Object set(Object object, String key) {
	        return mMemoryCache.put(key, object);
	    }
		  
	    public boolean delete(String key) {
	        mMemoryCache.remove(key);
	        return true;
	    }
	 
	    public void clear() {
	        mMemoryCache.evictAll();
	    }	  
	
	}

使用起来和我们的`Redis`一样，用一个`get`和`set`函数来保存和缓存数据。
他的运用场景也类似Redis，主要就是`缓存数据`，提高数据获取速度，例如来自数据库的数据。
突然觉得下次有空应该写多一个库，像`Spring`的`@Cache`那样，基于注解的使用这个缓存机制！
感觉挺酷酷的，不知道有没人写过了。加个todo 

	// TODO 写个基于注解的缓存库


# 起航
看了这个基本的使用案例，我们也很熟悉的感觉到一件事情，我们常用的Universal-Image-Loader库好像也有这个的实现版本，我们使用它的时候的配置选项，里面有一个`LruMemoryCache`

	ImageLoaderConfiguration config = new ImageLoaderConfiguration.Builder(this)
	                .threadPoolSize(3) // default 3
	                .threadPriority(Thread.NORM_PRIORITY - 1)
	                .memoryCache(new LruMemoryCache(3 * 1024 * 1024)) // <-看这里
	                .memoryCacheSizePercentage(13) 	
	                .build();
这是机会，我们去看下他写的内容！当然他还有很多个版本的缓存配置，有兴趣慢慢都看完他
路径是：

	package com.nostra13.universalimageloader.cache.memory.impl;

整个类就一百多行，我们直接粘贴上来，有个整体的印象，再做细看

	public class LruMemoryCache implements MemoryCache {
	
		private final LinkedHashMap<String, Bitmap> map;
	
		private final int maxSize;
		/** Size of this cache in bytes */
		private int size;
	 
		public LruMemoryCache(int maxSize) {
			if (maxSize <= 0) {
				throw new IllegalArgumentException("maxSize <= 0");
			}
			this.maxSize = maxSize;
			this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
		}
	 
		@Override
		public final Bitmap get(String key) {
			if (key == null) {
				throw new NullPointerException("key == null");
			}	
			synchronized (this) {
				return map.get(key);
			}
		}
	 
		@Override
		public final boolean put(String key, Bitmap value) {
			if (key == null || value == null) {
				throw new NullPointerException("key == null || value == null");
			}
	
			synchronized (this) {
				size += sizeOf(key, value);
				Bitmap previous = map.put(key, value);
				if (previous != null) {
					size -= sizeOf(key, previous);
				}
			}
	
			trimToSize(maxSize);
			return true;
		}
	 
		private void trimToSize(int maxSize) {
			while (true) {
				String key;
				Bitmap value;
				synchronized (this) {
					if (size < 0 || (map.isEmpty() && size != 0)) {
						...
					}
	
					if (size <= maxSize || map.isEmpty()) {
						break;
					}
	
					Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
					if (toEvict == null) {
						break;
					}
					key = toEvict.getKey();
					value = toEvict.getValue();
					map.remove(key);
					size -= sizeOf(key, value);
				}
			}
		}
	 
		@Override
		public final Bitmap remove(String key) {
			if (key == null) {
				throw new NullPointerException("key == null");
			}
	
			synchronized (this) {
				Bitmap previous = map.remove(key);
				if (previous != null) {
					size -= sizeOf(key, previous);
				}
				return previous;
			}
		}
	
		@Override
		public Collection<String> keys() {
			synchronized (this) {
				return new HashSet<String>(map.keySet());
			}
		}
	
		@Override
		public void clear() {
			trimToSize(-1); // -1 will evict 0-sized elements
		}
	 
		private int sizeOf(String key, Bitmap value) {
			return value.getRowBytes() * value.getHeight();
		}
	
		@Override
		public synchronized final String toString() {
			return String.format("LruCache[maxSize=%d]", maxSize);
		}
	}


# 详解

看下构造函数

	public LruMemoryCache(int maxSize) {
		if (maxSize <= 0) {
			throw new IllegalArgumentException("maxSize <= 0");
		}
		this.maxSize = maxSize;
		this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
	}
我们看到他是怎么保存数据的
 
	this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
	
用的是`LinkedHashMap<String, Bitmap>` ，而且`initialCapacity`居然是0，作者看来为了节约空间，一开始都初始化为0，不想浪费空间啊。不过需提一下的是，虽然扩展容量时，用`初始容量*加载因子`来确定下次的容量，而我们的初始容量为**0**，加载因子为**0.75**，不过我们的HashMap的底层实现上还是会初始化为`1`作为最小值的。

关于这个结构的一个特点就是  **链表中元素的顺序可以按访问顺序(调用get方法)排列**，如果我们的第三个参数是`TURE`的话。

> rue if the ordering should be done based on the last access 
> (from  least-recently accessed to most-recently accessed), 
> and false if the ordering should be the order in which the entries were inserted.

所以我们可以感觉就像优先队列那样，那个被掉调用了，就让他排前面。
感觉这些数据结构应该下次有空也温习下，因为有些结构是线程安全适合高并发的环境，在一般的安卓开发用得也不算多了，都生疏了，不像当年写后台那样要求那么高（其实就是懒了，不求性能最优）。

获取数据的方式也很直接，直接拿

	public final Bitmap get(String key) {
	       ...
	 		synchronized (this) {
				return map.get(key);
			}
		}

对于put函数，放进去后，不管如何都去执行下压缩函数，感觉可以加多个判断避免调用啊，我们看下trimToSize里面的内容。  

	public final boolean put(String key, Bitmap value) {
		if (key == null || value == null) {
			throw new NullPointerException("key == null || value == null");
		}

		synchronized (this) {
			size += sizeOf(key, value);
			Bitmap previous = map.put(key, value);
			if (previous != null) {
				size -= sizeOf(key, previous);
			}
		}

		trimToSize(maxSize);
		return true;
	}
	
	private int sizeOf(String key, Bitmap value) {
		return value.getRowBytes() * value.getHeight();
	}


这个函数主要执行的就是看下有没超过限制的最大大小，超了的就拿最少使用的那个开刀，也就是排在最后面那个。感觉平时也应该混个脸熟啊，要不然像这样，成为了被T的沉默羔羊。

	private void trimToSize(int maxSize) {
		while (true) {
			String key;
			Bitmap value;
			synchronized (this) {
				if (size < 0 || (map.isEmpty() && size != 0)) {
					throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
				}
	
				if (size <= maxSize || map.isEmpty()) {
					break;
				}
	
				Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
				if (toEvict == null) {
					break;
				}
				key = toEvict.getKey();
				value = toEvict.getValue();
				map.remove(key);
				size -= sizeOf(key, value);
			}
		}
	}
好了，其余的没什么好看的了。
我们转去看下安卓的实现版本是怎样的

# 安卓的实现版本

很开心的告诉你，他们两者高度雷同，用的也是这个`LinkedHashMap`，不过有一点不一样的是一些细节上的。
我们来看下他的get函数

	public final V get(K key) {
	        if(key == null) {
	            throw new NullPointerException("key == null");
	        } else {
	            Object mapValue;
	            synchronized(this) {
	                mapValue = this.map.get(key);
	                if(mapValue != null) {
	                    ++this.hitCount;
	                    return mapValue;
	                }
	
	                ++this.missCount;
	            }
	
	            Object createdValue = this.create(key);
	            if(createdValue == null) {
	                return null;
	            } else {
	                synchronized(this) {
	                    ++this.createCount;
	                    mapValue = this.map.put(key, createdValue);
	                    if(mapValue != null) {
	                        this.map.put(key, mapValue);
	                    } else {
	                        this.size += this.safeSizeOf(key, createdValue);
	                    }
	                }
	
	                if(mapValue != null) {
	                    this.entryRemoved(false, key, createdValue, mapValue);
	                    return mapValue;
	                } else {
	                    this.trimToSize(this.maxSize);
	                    return createdValue;
	                }
	            }
	        }
	    }
	

	protected V create(K key) {
	        return null;
	    }

但没有命中的时候，他会去调用哪个`create`函数，不过默认的返回的是空。

接着我们来看下，我们的put函数

	public final V put(K key, V value) {
	        if(key != null && value != null) {
	            Object previous;
	            synchronized(this) {
	                ++this.putCount;
	                this.size += this.safeSizeOf(key, value);
	                previous = this.map.put(key, value);
	                if(previous != null) {
	                    this.size -= this.safeSizeOf(key, previous);
	                }
	            }
	
	            if(previous != null) {
	                this.entryRemoved(false, key, previous, value);
	            }
	
	            this.trimToSize(this.maxSize);
	            return previous;
	        } else {
	            throw new NullPointerException("key == null || value == null");
	        }
	    }
这里要说的是哪个`entryRemoved`是空的，我们可以重写做点什么？

	protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {
	}

# 后记

没什么好说的

