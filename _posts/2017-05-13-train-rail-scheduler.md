---
layout: post
title: 单向火车轨道调度
date: 2017-05-13 14:47:06
categories: Java
---

# 单向火车轨道调度
需求来源：飘屏弹幕从屏幕的一边移动到屏幕的另一端，弹幕只有距离屏幕边缘一段距离后，这条轨道上才能允许播放下一条弹幕。

屏幕方向变化（横竖屏切换）时，轨道数目会发生变化。轨道数目减少时，被隐藏的轨道上运行的弹幕依然会运行，只是会被隐藏，等到屏幕方向变化，轨道数目增加时，继续显示。

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;

import rx.Observable;
import rx.functions.Action1;

/**
 * 当一条铁轨上没有火车或者最后面的火车已开出一段安全距离，则认为该铁轨不繁忙，可以安排火车上路。
 * 
 * @author rongzhisheng
 *
 */
public class Scheduler {
    private int speed = 2;// 火车运行速度，单位米/秒
    private int safeDistanse;// 两辆火车之间允许的最小距离，即安全距离，单位米
    private int railCount;// 铁轨数量，可变
    private Map<Integer, Boolean> railBusyMap;// 铁轨是否空闲，铁轨下标从0开始

    public Scheduler(int safeDistanse, int railCount) {
        this.safeDistanse = safeDistanse;
        this.railCount = railCount;
        railBusyMap = new ConcurrentHashMap<>();
    }

    public void setRailCount(int railCount) {
        if (railCount < 0) {
            println("invalid railCount " + railCount);
            return;
        }
        this.railCount = railCount;
    }

    /**
     * 为新车安排一条铁轨
     * 
     * @param specifyRailIndex
     *            指定了铁轨编号<br>
     *            小于0的值表示不指定 [0, railCount)
     * @return -1表示无法安排
     */
    public int scheduleRail(int specifyRailIndex) {
        final int finalRailCount = railCount;
        if (finalRailCount <= 0 || specifyRailIndex >= finalRailCount) {
            return -1;
        }

        if (specifyRailIndex >= 0) {
            Boolean busy = railBusyMap.get(specifyRailIndex);
            if (busy == null || busy == false) {
                railBusyMap.put(specifyRailIndex, true);
                return specifyRailIndex;
            }
            return -1;
        }

        for (int i = 0; i < finalRailCount; i++) {
            Boolean busy = railBusyMap.get(i);
            if (busy == null || busy == false) {
                railBusyMap.put(i, true);
                return i;
            }
        }
        return -1;
    }

    /**
     * 占据一条轨道后调用此方法
     * 
     * @param railIndex
     * @param trainLength
     */
    public void depart(int railIndex, int trainLength) {
        if (railIndex < 0) {
            println("invalid rail " + railIndex);
            return;
        }
        Boolean busy = railBusyMap.get(railIndex);
        if (busy == null || busy == false) {
            println("not occupy rail " + railIndex);
            return;
        }
        println("depart on rail " + railIndex);
        int totalLength = trainLength + safeDistanse;
        // 单位：秒
        int busyTime = totalLength / speed;
        Observable.timer(busyTime, TimeUnit.SECONDS).subscribe(
                new Action1<Long>() {
                    @Override
                    public void call(Long l) {
                        railBusyMap.put(railIndex, false);
                        println("free rail " + railIndex);
                    }
                });
    }

    /**
     * 根据给定参数获得特定轨道的margin
     * 
     * @param railIndex
     *            轨道索引，0轨道在最上方
     * @param totalRailWidth
     *            最上方轨道到最下方轨道间的距离
     * @param trainWidth
     *            每条轨道的宽度
     * @param ifMust
     *            如果找不到轨道，是否放弃
     * @return
     */
    public int getMargin(int railIndex, int totalRailWidth, int trainWidth,
            boolean ifMust) {
        final int finalRailCount = railCount;
        if (railIndex < 0 || finalRailCount <= 0) {
            return -1;
        }
        // 刚分配好一个轨道，结果轨道数变少了
        if (railIndex >= finalRailCount) {
            if (ifMust) {
                return -1;
            }
            railIndex = scheduleRail(-1);
            return getMargin(railIndex, totalRailWidth, trainWidth, ifMust);
        }

        if (totalRailWidth < trainWidth) {// 部分显示
            return 0;
        } else {
            return (totalRailWidth - trainWidth) / (finalRailCount - 1)
                    * railIndex;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Scheduler s = new Scheduler(4, 2);
        int railIndex = s.scheduleRail(-1);
        s.depart(railIndex, 4);
        railIndex = s.scheduleRail(-1);
        s.depart(railIndex, 4);

        // 无法分配轨道
        railIndex = s.scheduleRail(-1);
        s.depart(railIndex, 4);

        Thread.sleep(5_000);
        // 找到轨道
        railIndex = s.scheduleRail(0);
        s.depart(railIndex, 4);

        // 缩减轨道
        s.setRailCount(1);

        railIndex = s.scheduleRail(1);
        s.depart(railIndex, 4);

        Thread.sleep(2_000);
        // 增加轨道
        s.setRailCount(3);

        railIndex = s.scheduleRail(-1);
        s.depart(railIndex, 4);

        railIndex = s.scheduleRail(-1);
        s.depart(railIndex, 4);

        railIndex = s.scheduleRail(-1);
        s.depart(railIndex, 4);

        int rail = 0;
        println("rail " + rail + " margin: "
                + s.getMargin(rail++, 10, 2, false));
        println("rail " + rail + " margin: "
                + s.getMargin(rail++, 10, 2, false));
        println("rail " + rail + " margin: "
                + s.getMargin(rail++, 10, 2, false));
        println("rail " + rail + " margin: "
                + s.getMargin(rail++, 10, 2, false));

        Thread.sleep(6_000);
        println("END");
    }

    private static void println(Object obj) {
        SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
        System.out.println(sdf.format(new Date()) + ": " + obj);
    }

}

```