---
layout: post
title: 状态机
date: 2017-05-20 10:31:46
categories: Java
---
# 状态机

本文参考[这篇文章](http://www.cnblogs.com/itTeacher/archive/2012/12/04/2801597.html)

最近项目需要接入一个新VoIP引擎，先前使用其他引擎时遇到接入方（使用我们封装的SDK的程序，以下简称为`客户端程序`）按照令人意想不到的顺序来调用我们的程序（以下简称为`SDK`）接口的现象。

我以前也看过Android中的MediaPlayer的状态转化图，认为应该给我们的程序添加一个状态机来限定客户端程序的行为，达到保护引擎的目的。

由于引擎内部是有状态的（有记忆效应），我的要求是客户端程序在调用SDK的接口时，SDK需要根据内部的状态来告知客户端程序该操作可不可以进行，以及操作是否成功（如果可以进行，当且仅当操作成功的情况下才迁移到下一个状态）。

Context接口定义暴露给客户端程序的接口

State接口里的方法对应于Context里的方法，Context会把调用委托给State

下面演示了一个进出房间的状态转换

```java
public interface Context {
    State getState();

    void setState(State state);

    boolean joinRoom();

    boolean leaveRoom();
}

class ContextImpl implements Context {
    private State currentState;

    public ContextImpl() {
        currentState = new OutdoorState();
    }

    @Override
    public State getState() {
        return currentState;
    }

    @Override
    public void setState(State state) {
        currentState = state;
    }

    @Override
    public boolean joinRoom() {
        return currentState.handleJoinRoom(this);
    }

    @Override
    public boolean leaveRoom() {
        return currentState.handleLeaveRoom(this);
    }

}
```

```java
public interface State {
    String getName();

    boolean handleJoinRoom(Context context);

    boolean handleLeaveRoom(Context context);
}

class OutdoorState implements State {

    @Override
    public String getName() {
        return "房间外";
    }

    @Override
    public boolean handleJoinRoom(Context context) {
        System.out.println("准备进入房间");
        context.setState(new IndoorState());
        return true;
    }

    @Override
    public boolean handleLeaveRoom(Context context) {
        System.out.println("当前在房间外，不能离开房间");
        return false;
    }

}

class IndoorState implements State {

    @Override
    public String getName() {
        return "房间内";
    }

    @Override
    public boolean handleJoinRoom(Context context) {
        System.out.println("当前在室内，不能进入房间");
        return false;
    }

    @Override
    public boolean handleLeaveRoom(Context context) {
        System.out.println("准备离开房间");
        context.setState(new OutdoorState());
        return true;
    }

}
```

上面例子在State的各种实现里，只是输出一句话，然后就设置了下一个状态。实际工程中，要进行一项具体的工作（这项工作可能由其他类来完成，在我的需求里，是被保护起来的引擎），并且根据工作的返回结果是否成功来决定是否设置下一个状态以及返回值。

如果引擎不是单例（这很少见）并且初始化在ContextImpl里，那么相应的方法需要增加一个引擎类型的参数；相比之下，把引擎做成单例，通过类名到处可以引用到，就可以不必增加这个额外的参数。