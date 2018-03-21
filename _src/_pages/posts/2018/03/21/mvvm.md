---
title: MVVM 架构模式与其背后的设计思想
layout: post
format: markdown
type: post
tags: android,design pattern
timestamp: 1521647456
---

在去年 Google 引入 Android Architecture Components 之前，Google 在 Android 原生应用开发的架构上并没有提出任何参考设计，但在江湖上早有各种 MVC、MVP 和 MVVM 的应用。这篇文章主要介绍 MVVM 以及应用这种架构模式背后所蕴含的设计思想。

架构的意义可以被总结为四个字 “抵御风险”，这里风险可以理解为软件开发周期里面的各类成本，总结起来其实就是时间成本和人力成本，因为不管是开发、维护、管理或者计划都可以算作人力和时间的投入。架构好坏的评判标准就在于其抵御风险的能力。好的架构需要具备良好的扩展性、可伸缩性、可维护性、易用性、灵活性（快速适应变化的能力）、降低或者消除重复的能力等等。

MVVM，其核心在于解耦，解耦的目的是为了让我们能够实现真正意义上的模块化开发，进而达到以上提到的架构优点。以往我们在 Activity 里面实现了数据请求、数据处理逻辑、更新 UI 等等，并且认为一个 Activity 相对于整个 App 来说是一个模块，但这只是在代码组织上粒度很粗的模块，并没有考虑区分里面包含的各种不同语义的代码，其复用性和可维护性都是非常差的，一个良好定义的模块应该具备清晰的代码组织和语义单一的结构，一个良好架构的系统就是通过组合这些模块来实现的。

MVVM 是一种设计的指导思想，对于里面的解耦方式并没有规定具体实现，但通常都会采用观察者模式，有的实现了单向解耦（比如本文中的例子），有的实现了双向解耦（比如 WPF）。在本文中，我们实现的是单向解耦，即 View 依赖 ViewModel，但 ViewModel 不依赖 View。如下图：

![mvvm](/image/mvvm.png)

在深入解析上面这个图之前，我先介绍一下在例子中将用到的一个叫 ValueBinding 的组件，这个组件只是一个非常简单的观察者模式的实现，可被用于封装 Android 中的 View 控件（如 TextViewBinding）用于接收来自 ViewModel 层的数据更新，从而解耦 ViewModel 层和 View 层。以 TextViewBinding 为例，实现类似这样：

    public class ValueBinding<T> {  
      private ValueProvider<T> mValueProvider;  
      private ValueConsumer<T> mValueConsumer;  
      private T mValue;  

      public void consume(ValueConsumer<T> consumer) {  
        mValueConsumer = consumer;  
      }  
      public void provide(ValueProvider<T> provider) {  
        mValueProvider = provider;  
      }  

      public void set(T value) {  
        if (mValueConsumer != null) {  
          mValueConsumer.consume(value);  
        }  
        mValue = value;  
      }  
      public T get() {  
        if (mValueProvider != null) {  
          mValue = mValueProvider.provide();  
        }  
        return mValue;  
      }  

      public interface ValueProvider<T> {  
        T provide();  
      } 
      public interface ValueConsumer<T> {  
        void consume(T value);  
      }  
    }

    public class TextViewBinding extends ValueBinding<CharSequence> {  
      public TextViewBinding(TextView textView) {  
        super();  
        provide(() -> textView.getText().toString());  
        consume(data -> textView.setText(data));  
      }  
    }

从上图中可以看出，View 依赖 ViewModel，根据接收到的用户事件，执行 ViewModel 提供的功能。ViewModel 不依赖 View，不持有任何 View 层控件的引用，ViewModel 只持有来自 View 层的 ValueBinding，ValueBinding 背后的具体实现可以是 TextViewBinding、ImageViewBinding、甚至 RecyclerViewBinding，但 ViewModel 不需要了解它们的具体实现，只需要知道他们都需要「数据」，ViewModel 依赖 Model 层提供的数据访问功能，其本身只实现「数据获取/提交、数据转换、数据绑定」的功能，不涉及任何操作「更新 UI」的代码。

下面以一个简单的例子展示 View 与 ViewModel 的实现：

    public class TestView extends Fragment {
      private TestViewModel mViewModel;
      public void onViewCreated(...) {
        ...
        // 绑定 View 到 ViewModel 中定义的一个 key
        mViewModel.bind(TestViewModel.DATA_TEST,
          new TextViewBinding(mTvContent));
        mViewModel.loadDataAsync();
      }
    }
    public class TestViewModel {
      public final static String DATA_TEST = "key_data_test";
      public void bind(String key, ValueBinding binding) {
        ...
      }
      public void loadDataAsync() {
        loadDataFromRemote(data -> {
          getBinding(DATA_TEST).set(data); // 绑定数据
        });
      }
    }

通过 ValueBinding 我们把「刷新 UI 控件」这个问题 generalize 成了定义更泛的「绑定数据」问题，在一定程度上 ViewModel 就是在解决一个更抽象的「数据加工和数据绑定」的问题了，虽然这个问题仍然是某一领域的具体问题（比如更新用户名），但至少它已经不需要关心 ValueBinding 背后的实现是一个 View 还是单元测试里面的各种模拟输入了。这样做有以下优点：

1. 实现业务逻辑的复用；
2. 降低了代码的复杂度；
3. 实现线上同一份数据应用到不同 UI 的 AB Testing；
4. 可对业务逻辑代码进行单元测试；
5. 团队协作的分工可以更细，比如有人实现 UI，有人实现业务逻辑。

软件的本质其实只是一个数据的转换器（Transformer），它是一个黑盒，它把来自外界的输入通过内部的处理加工（转换）成软件使用者希望看到的样子（输出）。比如我们把一个新闻客户端当成一个转换器，点击一个新闻标题，我们就看到了这篇新闻的内容，这里输入是「点击」，输出是「新闻内容」，新闻客户端在这个输入和输出中间可能做了很多事情，比如新闻标题对应的是一个新闻内容的网址，但新闻客户端并不会经过一个中间页面把网址显示给你看告诉你接下来要通过这个地址去下载内容了，这些用户不感知也不关心，它所做的是在背后直接把内容下载回来并显示出来。这就是「充分抽象」的结果。作为软件开发者，当我们在开发一个软件的时候，我们理应把里面每个模块设计成一个个「充分抽象」的小黑盒，比如：「A 模块依赖 B 模块，那么 B 模块就不应该让 A 模块感知任何与 B 模块输出无关的任何设计或实现细节」。

结合 ValueBinding 和上面对模块设计的理解，我们可以拿 Android 中的 RecyclerView 来举例，对其进一步封装，令其达到「充分抽象」的效果。RecyclerView 是一个支持显示一组数据的 UI 控件，通常用来实现列表数据显示，它需要一个 Adapter 用于提供数据和一组 ViewHolder 用于复用列表中的 item view 以提高渲染性能。这些对于熟悉 Android 的开发者来讲，听起来很合理，没什么不对，但这是一个「没有充分抽象」的例子。RecyclerView 的输入是「列表数据」，输出是「展示列表数据的界面」，但中间的 Adapter 和 ViewHolder 本不该让「模块使用者」感知，因为这些并不影响输出。对于 RecyclerView，我们可以实现 RecyclerViewBinding 对这些中间步骤进行封装，最终达到以下效果来实现列表数据展示的功能：

    public class RecyclerViewBinding<T>
      extends ValueBinding<List<T>> {
      public static class Builder {
        public void bindItem(
          @IdRes int itemLayout, DataBinder binder) {
          ...
        }
      }
    }

    ValueBinding recyclerViewBinding =
         new RecyclerViewBinding.Builder()
      .recyclerView(recyclerView)
      .<Data1>bindItem(R.layout.item1, (dataBinder, data1) -> {
        dataBinder.set(R.id.someTextView, data1.someText);
      })
      .<Data2>bindItem(R.layout.item1, (dataBinder, data2) -> {
        dataBinder.set(R.id.someTextView, data2.someText);
      })
      .build();

    // 在 ViewModel 里面，我们只需要执行以下代码即可
    recyclerViewBinding.set("data", listOfData1AndData2);

上面的代码没有给出 RecyclerViewBinding 的具体实现，给出的是其提供的关键接口与「充分抽象」之后的使用方法。RecyclerViewBinding 对于使用者 ViewModel 来讲，只需要通过其 set() 方法提供 listOfData1AndData2 即可，至于这些数据会被怎么显示不是它需要关注的。对于 View 层来讲，它也是 RecyclerViewBinding 的使用者，这里 View 只需要关注列表里面每个 item 的样式（布局文件），每个 item 需要显示什么「类型」的数据（Data1 和 Data2），并把每个 item 中的子 view 与对应的数据关联起来，仅此而已，不需要了解 Adapter 也没有 ViewHolder 这些概念。

通过上面「充分抽象」的例子，可以看出它带来的好处，它让我们在使用模块或组件的时候「只关注必须关注的细节」，或更具体一点，「只需关注输入和输出，不需要了解过程」。关注点少了，实现必然更简单。

再回到 MVVM 的设计，它主张的分层和解耦，其实都是「充分抽象」的一种体现，无非就是通过各种手段（如观察者模式或者其他数据绑定技术）让每一个模块（View 和 ViewModel）变得「更纯粹」，只实现他们应该实现的功能。

这也是为什么最近几年各种命令式语言（Imperative Programming Language）如 C++/Java/Javascript 都引入了函数式编程的重要原因，因为 Pure，因为「纯粹」的设计才是设计本来该有的样子。
