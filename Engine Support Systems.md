Contents:
-  [[#Subsystem Start-up and Shut-down]]
-  [[#Memory Management]]
-  [[#Containers]]
-  [[#Strings]]
-  [[#Engine Configuration]]
# Subsystem Start-up and Shut-down

The independence tells us the order in which subsystem must star-up and shutdown. In C++ we can do it with `constructors` and `destructors` however most one implements such systems as `global` or `static` singleton variables. For them C++ doesn't guarantee the order of initialization and shutting down. They are shut-down only after `main()` or `WinMain()`  returns. This isn't what we need.  

If there was a way to do  it, we would write:
```cpp
class RenderManager
{
	public:
		RenderManager()
		{
			// Start up the manager...
		}
		~RenderManager()
		{
			// shut down the manager...
		}
};

// singleton instance
static RenderManager gRenderManager;
```

## Construction on demand
If a `static` variable is declared inside the function then it is created only after that function is called and not before `main`. It gives us control on when and how to create our singleton class. But it doesn't guarantee the order of destruction of singletons. However below is the code: 
```cpp
class RenderManager
{
	public:
		static RenderManager &get()
		{
			//This function-static will be constructed on the first call the
			// first call to this function
			static RenderManager* gpSingleton = nullptr;
			if(gpSingleton == nullptr)
			{
				gpSingleton = new RenderManager;
			}

			ASSERT(gpSingleton);
			return *gpSingleton;
		}
		RenderManager()
		{
			// start up other managers we depend on by calling thier get() functions
			VideoManager::get();
			TextureManager::get();
			 // Now start up the render manager
		}
		~RenderManager()
		{
			// shut down the manager...
		}
};
```

## Simple Approach that works

A simple "brute force" method that works is to manually mention how to start and shut down the global static singletons and ensure that Constructors and Destructors do nothing. 

```cpp
class RenderManager
{
	public:
		RenderManager()
		{
			// Do nothing
		}
		
		~RenderManager()
		{
			// Do nothing
		}
		
		void StartUp()
		{
			// start up the manager
		}
		
		void ShutDown()
		{
			// Shut Down the manager
		}
		
		// ...
}

class PhysicsManager { /* similarly ... */ };
class AnimationManager { /* similarly ... */ };
class MemoryManager { /* similarly ... */ };
class FileSystemManager { /* similarly ... */ };

// ...

RenderManager gRenderManager;
PhysicsManager gPhysicsManager;
AnimationManager gAnimationManager;
VideoManager gVideoManager;
MemoryManager gMemoryManager;
FileSystemManager gFileSystemManager;

// ...

int main(int arg, const char* argv)
{
	// start up engine systems in the correct order
	gMemoryManager.StartUp();
	gFileSystemManager.StartUp();
	gVideoManager.StartUp();
	gRenderManager.StartUp();
	gAnimationManager.StartUp();
	gPhysicsManager.StartUp();
	
	// .. 
	
	// Run the game
	
	// Shutdown everything in the reverse order
	gPhysicsManager.ShutDown();
	gAnimationManager.ShutDown();
	gRenderManager.ShutDown();
	gVideoManager.ShutDown();
	gFileSystemManager.ShutDown();
	gMemoryManager.ShutDown();
	gSimulationManager.ShutDown();
	
	return(0);
}
```

There are more "elegant ways" to do this such as using priority queue or by using graph and asking each manager what it depends upon and then traverse this dependency graph to figure out the most optimal way to initialize these managers but the brute force way always wins. 

- Simple
- Explicit
- Easy to debug

## Some Examples from Real Engines
### OGRE

In OGRE there is a single singleton object `OGRE::Root` that manages all the creation and destruction of managers to it's easy for programmers to just use `new OGRE::Root`

```cpp

class _OgreExport Root : public Singleton<Root>
{
	// ...
	
	// Singletons
	LogManager* mLogManager;
	ControllerManager* mControllerManager;
	SceneManagerEnumerator* mSceneManager; 
	SceneManager* mSceneManager;
	DynLibManager* mDynLibManager;
	ArchiveManager* mArchiveManager;
	// List other managers
	
	// ... 
};

Root::Root(
	const String& pluginFileName,
	const String& configFileName,
	const String& logFileName
)
{
	// superclass will do singleton checking
	String msg;
	
	// init
	mActiveRenderer = 0;
	
	mVersion 
		= StringConverter::toString(OGER_VERSION_MAJOR)
		+ "."
		+ StringConverter::toString(OGER_VERSION_MINOR)
		+ "."
		+ StringConverter::toString(OGER_VERSION_PATCH)
		+ OGER_VERSION_SUFFIX + " " 
		+ "(" + OGER_VERSION_NAME + ")";
		
	mConfigFileName = configFileName;
	
	// create log manager and default log file if there is no log manager yet
	
	if(LogManager::getSingletonPtr() == 0)
	{
		mLogManager = new LogManager();
		mLogManager->createLog(logFileName, true, true);
	}
	
	// dynamic manager
	mDynLibManager = new DynLibManager();
	mArchiveManager = new ArchiveManager();
	
	// Resource Manager
	mResourceManager = new ResourceManager();
	
	// and so on ... 
}
```

### Naughty Dog's Uncharted and The Last of Us Series

Here, engine start-up is not a simple sequence of allocating singleton instances but a wide range on OS services and third party libs are initialized. Also dynamic memory allocation is avoided where possible. 

```cpp
Err BigInit()
{

	init_exception_handler();
	U8* pPhysicsHeap = new(kAllocGlobal, kAligh16) U8[ALLOCATION_GLOBAL_PHYS_HEAP];
	PhysicsAllocationInit(pPhysicsHeap, ALLOCATION_GLOBAL_PHYS_HEAP);
	
	g_textDb.Init();
	g_textSubDb.Init();
	g_spuMgr.Init();
	
	g_drawScript.InitPlatform();
	
	PlatformUpdate();
	
	thread_t init_thr;
	thread_create(&init_thr, threadInit, 0, 30, 64*1024, 0, "Init");
	
	char masterConfigFileName[256];
	snpritnf(masterConfigFileName, sizeof(masterConfigFileName), MASTER_CFG_PATH);
	{
		Err err = ReadConfigFromFile(masterConfigFileName);
		if(err.Failed())
		{
			MsgErr("Config file not found (%s).\n", masterConfigFileName);
		}
	}
	
	memset(&g_discInfo, 0, sizeof(BootDiscInfo));
	int err1 = GetBootDiscInfo(&g_discInfo);
	Msg("GetBootDiscInfo() : 0x%x\n", err1);
	if(err1 == BOOTDISCINFO_RET_OK)
	{
		printf("titleId : [%s]\n", g_discInfo.titleId);
		printf("parentalLevel : [%s]\n", g_discInfo.parentalLevel);
	}
	
	g_fileSystem.Init(g_gameInfo.m_onDisc);
	g_languageMgr.Init();
	if(g_shouldQuit) return Err::kOK;
	
	// and so on...
}

```

# Memory Management

The speed of a program is dictated by how it uses RAM. 
- Dynamic memory allocations like `malloc` and `new` are expensive.
- On modern CPU small contiguous memory access and good memory access patterns are good and access small chunks on memory on wide address results in performance hits.

Malloc, free, new and delete need kernel mode and so results in context switching from user to kernel thread this is quit expensive and need to be avoided. But no Game Engine can avoid dynamic memory allocations so the solution is to use Custom Memory Allocators. They can allocate a huge chunk of memory at once and minimizing user-kernel thread switching as well as make several assumptions about memory access patterns for which we can optimize it. 

[Further Reading](http://www.swedishcoding.com/2008/08/31/Are-we-out-of-memory)

# Custom Allocators
### Stack-Based Allocators
![[Pasted image 20250911110751.png]]

```cpp
class StackAllocator
{
	public:
		// Stack marker: Represents the current top of the stack
		// You can only roll back to a marker, not to arbitrary 
		// location within the stack
		typedef U32 Marker;
		
		// Constructs a stack allocator with the given total size
		explicit StackAllocator(U32 stackSize_bytes);
		
		// Allocates a new block of the given size from stack top
		void* alloc(U32 size_bytes);
		
		// Returns the marker to the current stack top
		Marker getMarker();
		
		// Rolls the stack back to the prevous marker
		void freeToMarker(Marker marker);
		
		// clear entier stack (rolls back to zero)
		void clear();
};
```

### Double Ended Stack Allocators
![[Pasted image 20250911111927.png]]

### Pool Allocators
__NOTE:__ Read the blog by Ginger Bill.

* Use this when you want to allocate same sized elements. 
* In pool allocator we maintain a linked list of free memory blocks. 
* These are free blocks in the stack allocator. 
* If the element size >= sizeof(void*) then each element of free list can store the link to the next element. 
* When free element is allocated remove it from the list and when its free insert it back. 
* Both Allocations and free are O(1) operations.

### Aligned Allocators

// TODO (yash): 

## Fragmentation

Fragmentation is not an issue in OS that support virtual memory as the fragmentated memory is chucked together by the OS in virtual address space and makes it look like it's contiguous. But many embedded systems don't support virtual memory. 

### Avoiding Fragmentations with Stack and Pool Allocators
![[Pasted image 20250911115743.png]]

* Fragmentation is impossible in stack allocators
* There is fragmentations in pool allocators but it doesn't bother us because the size of each block is same and it wont result in not having a block of enough size.
### Defragmentation and Relocation

When variable sized allocations are made in random order and freed in random order, this results in fragmentation of heap, leaving holes in it. On way to deal with this is to "bubble up" the holes. That is whenever a new allocation is made look if there are holes in the below address of this allocation, if so shift it below and this will result it bubbles coming on top and bottom having contiguously allocated blocks of memory. 
![[Pasted image 20250911114116.png]]

Implementing this is not tricky, but this method invalidates the pointers to previous allocations as their addresses has been shifted. There are two ways to deal with this:   
1. __Handles__
	Handles are index into a non-relocatable table of pointers. Whenever shifting of addresses is done, the table is updated with new addresses but their indices won't change
2. __Smart Pointers__
	Smart Pointers are object that wraps traditional pointers. One can now write appropriate methods to handle fragmentations. One way is to use linked list and whenever the addresses are shifted, pointers are adjusted appropriately. 

But sometimes we can't relocate memory, say third party system doesn't uses smart pointers then arrange memory for this lib from a special buffer outside relocatable memory area, or if the fragmentation is in traces then just ignore it. 

__Amortizing Defragmentation Costs__

Defragmentation can be costly to be done on one large chunk of memory, instead divide this chunk into smaller pieces and limit the number of blocks that be shifted so N (8/16) .As long as allocations/deallocations doesn't happen faster than defragmentation there should be no hit on the frame rate. 
# Containers
Containers are nothing but data structures that are designed by programmers for handing data in their software. There are many examples of containers like:
- Arrays
- Linked List
- Graph
- Trees
- Dictionary, etc.

In order to traverse these containers we have to define a friend class termed "iterator". This helps us to keep the container implementation encapsulated and make traversing the data structure possible with a single loop. One can define an iterator for a graph and make DFS as simple as a single for loop. 

One should be cautioned about "post increment" and "pre increment". In post increment (++p) the value if first increased and then used in the expression. However this can result in stall as the CPU has to wait for the new incremented value. This will be even worse if the CPU pipeline is deep. However in cases of iterators for data structures often pre-increment (++p) is preferred as it takes time to copy the data structure into the expression.  

When designing the data structure first you have consider the memory access patterns of that data structure as well as Algorithmic Complexity of it's operations. 

There are several advantages of building our own DS:
- Total Control
- Opportunities of optimization
- Customizability
- Elimination of external dependencies
- Control over concurrent data structures 
	- When writing a container class, you have full control over the means by which its protected against concurrent access on multithreaded or multicore system

But before building our own data structure one should consider using the currently available ones provided by various libraries:
- C++ STL: 
	- Data hog
	- cryptic header
	- generic so slower
	- templating is not flexible enough to allow custom allocators. 
	
- BOOST:
	- Boost software licensing
	- No guarantee, if bugs then you are on your own 
	- Large .lib files problematic when size constraints

There are others such as Folly and Loki. 

### Dynamic Arrays and Chunky Allocations

> In the beginning of the project you may not know the upper bound of your arrays/allocations so use dynamic arrays but by the end replace them with static with known upper bounds.

Create a data structure containing a normal array and the size limit of it. When going out of that limit increase the allocation size. Usually it never shrinks back. 

When size increases copy the array to new location and discard the previous one. This is result in invalid pointers and increased insertion cost when needed to copy.

### Dictionaries and Hash tables

Dictionaries are containers that store key value pairs, they are implemented using BST or hash tables. Here we will discuss about hash tables. In hash tables, key value pairs are stored in such a way where the key can access the value. The key is converted in an integer using `hashing` and then its modulo-ed with the size of the table. So if hash table has 10 slots then pairs with key 32 and 42 will be stored as same key location. This problem is called "collision"

#### Resolving Collisions
There are two ways to solve this problem
__1. Closed Tables__
	![[Pasted image 20250911140325.png]]
	In closed tables, if there is any collision then it resolved via "probing". Its basically an algorithm for searching free slots. It's complex and difficult to implement and imposes limit on size. But it doesn't require dynamic memory allocations and so console programmers use it.
	
__2. Open Tables__
	![[Pasted image 20250911140310.png]]
	In open tables, if there is an collision then the values are stored as linked list. This allows us the create dictionaries without any size limit but required dynamic memory allocations
#### Hashing

Mathematically, 

```
h = H(k);
i = h % N
```

A good hashing function is the one that deterministic (gives same output for the same input) and is reasonably fast. It should distribute the hash values evenly across the table in decrease the chances of collision. 

![[Pasted image 20250911140538.png]]
#### Closed Hash Tables

In closed hash tables when there is collision we perform probing. There are two ways for it:
* Linear Probing : Iterate the table linearly and fill where there is space. Take modulo N for wrapping around
- Quadratic probing: iterate the table following : ( $i\pm j^2$ ). Remember to module N. 
[Prime number table size and Quadratic probing yields better results](http://stackoverflow.com/questions/1145217/why-should-hash-functions-use-a-prime-number-modulus)

In recent years [__ROBIN HOOD HASHING__](https://www.sebastiansylvan.com/post/robin-hood-hashing-should-be-your-default-hash-table-implementation/) has gained a lot of popularity.

# Strings
# Engine Configuration
