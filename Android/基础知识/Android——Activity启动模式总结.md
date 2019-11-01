[toc]

# 概述
在默认情况下，当我们多次启动同一个Activity 的时候，系统会创建多个实例并把它们一一放入任务栈中， 当我们单击back键，会发现这些Activity会一一回退。 

我们可能会发现一个问题多次启动同一个Activity，系统会重复创建多个实例。

Android在设计的时候不可能不考虑到这个问题，所以它提供了启动模式来修改系统的默认行为。目前有四种启动模式: standard、 singleTop、 singleTask 和singleInstance。

# standard
**标准模式。**

这也是系统的默认模式。每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否存在。创建的实例的声明周期符合典型情况下的Activity的声明周期。

在这种模式下，谁启动了这个Activity，那么这个Activity就运行在启动他的Activity所在的栈中。

当我们用ApplicationContext 去启动standard 模式的Activity会报错，是因为standard模式的Activity默认会进入启动他的Activity栈，但是非Activity类型的Context并没有所谓的任务栈。解决这个问题的方法就是为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标志位。这样启动的时候就会开启一个新的任务栈。

# singleTop
**栈顶复用模式。**

这种模式下，如果新的Activity已经位于任务栈的栈顶，那么此Activity不会重新创建。同时它的onNewIntent方法会被回调，通过此方法的参数我们可以取出当前请求的信息。

# singleTask
**栈内复用模式。**

这种模式下，只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，系统会回调其onNewIntent方法。

该模式的Activity请求启动后，系统首相会寻找是否存在有此Activity需要的任务栈，如果不存在，就重新创建一个任务栈，然后创建实例并放入栈中。

如果存在所需的任务栈，这是要看是否存在他的实例，如果有实例，就把他调到栈顶并调用他的onNewIntent方法。如果不存在，就创建实例并放入栈中。

例子：

- 比如目前任务栈S1中的情况为ABC,这个时候Activity D以singleTask模式请求启动，其所需要的任务栈为S2，由于S2和D的实例均不存在，所以系统会先创建任务栈S2，然后再创建D的实例并将其入栈到S2。
- 另外一种情况，假设D所需的任务栈为S1，其他情况如上面例子1所示，那么由于S1已经存在，所以系统会直接创建D的实例并将其入栈到S1.
- 如果D所需的任务栈为S1,并且当前任务栈S1的情况为ADBC,根据栈内复用的)原则，此时D不会重新创建，系统会把D切换到栈顶并调用其onNewIntent方法，同时由于singleTask默认具有clearTop的效果,会导致栈内所有在D上面的Activity全部出栈，于是最终S1中的情况为AD。

# singleInstance
**单实例模式。**

这是一种加强的singleTask模式。他除了具有singleTask模式所有的特性外，还加强了一点，就是此种模式的Activity只能单独位于一个任务栈中。

比如A是singleInstance模式，当A启动后，系统会给他创建一个新的任务栈，然后独自在这个新的任务栈中。由于栈内复用的特性，后续的请求均不会创建新的Activity。          