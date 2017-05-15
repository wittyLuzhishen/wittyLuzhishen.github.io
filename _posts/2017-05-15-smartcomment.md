---
layout: post
title: 智能弹幕提取
date: 2017-05-15 09:48:09
categories: Java
---
# 智能弹幕提取

本文描述了一种在一定长度的弹幕窗口中找出出现次数超过某个阈值且出现次数最多的弹幕的算法

## 算法之一（在当前需求下速度较快）
```java
import static com.rong.test.smartcomment.FrequencyCommentTest.SMART_GRASP_COUNT;
import static com.rong.test.smartcomment.FrequencyCommentTest.SMART_GRASP_SAME;

import java.util.HashMap;
import java.util.LinkedList;
import java.util.Map;

import com.rong.test.common.PrintUtil;

public class CommentFrequency implements ICommentContainer {
    private static final int CAPACITY = SMART_GRASP_COUNT;
    private boolean isRecentFirst = true;
    private LinkedList<UserComment> queue;
    private Map<String, Integer> map;
    private int maxTime;
    private String topComment;
    private long beginTime;

    public CommentFrequency() {
        queue = new LinkedList<>();
        map = new HashMap<>();
    }

    public CommentFrequency(boolean isRecentFirst) {
        this();
        this.isRecentFirst = isRecentFirst;
    }

    @Override
    public void insert(UserComment newUserComment) {
        String comment = newUserComment.comment;
//        if (newUserComment.userId <= 0 || StringUtils.isBlank(comment)) {
//            return;
//        }

        final int size = queue.size();
		if (!queue.isEmpty()) {
			for (int i = 0; i < size; i++) {
                if (isEquals(newUserComment, queue.get(i))) {
                    return;
                }
            }
        }

        boolean isNeedAddCount = true;
        if (size >= CAPACITY) {
            UserComment headUserComment = queue.poll();
            String head = headUserComment.comment;
            if (!comment.equals(head)) {
                Integer i = map.get(head);
                if (i != null) {
                    if (head.equals(topComment)) {
                        maxTime = 0;
                    }
                    if (i <= 1) {
                        map.remove(head);
                    } else {
                        --i;
                        recordTop(head, i);
                        map.put(head, i);
                    }
                } else {
                    PrintUtil.println("error, map have not string:" + head);
                }
            } else {
                isNeedAddCount = false;
            }
        }

        queue.offer(newUserComment);
        if (isNeedAddCount) {// comment不等于head
			Integer i = map.get(comment);
			i = i == null ? 1 : ++i;
			recordTop(comment, i);
			map.put(comment, i);
		}

        if (maxTime >= SMART_GRASP_SAME) {
        	PrintUtil.println("cost time:" + (System.nanoTime() - beginTime));
        	PrintUtil.println("maxTime:" + maxTime + ", topComment:" + topComment);
        }
    }

	private boolean isEquals(UserComment l, UserComment r) {
		return l.userId == r.userId && l.comment.equals(r.comment);
	}

    private void recordTop(String comment, int i) {
        if (i > maxTime || (isRecentFirst && i == maxTime)) {
            maxTime = i;
            topComment = comment;
        }
    }

    public int getMaxTime() {
        return maxTime;
    }

    public String getTopComment() {
        return topComment;
    }

	@Override
	public void setBeginTime(long beginTime) {
		this.beginTime = beginTime;
	}

}

```

## 接口
```java
public interface ICommentContainer {
    void insert(UserComment comment);
    
    void setBeginTime(long beginTime);
}
```

## 测试
```java
package com.rong.test.smartcomment;

import java.util.Random;

import com.rong.test.common.PrintUtil;

public class FrequencyCommentTest {
	public final static int SMART_GRASP_COUNT = 5;
	public static final int SMART_GRASP_SAME = 3;

	public static void main(String... args) {
         UserComment[] comments = getRandomStringArray(100, 2, 2, 10);
        //UserComment[] comments = parseFrom("1:b,5:a,5:b,3:a,3:a,3:a,1:a,3:c,1:b,3:b"
        // + ",1:b,5:a,5:b,3:a,3:a,3:a,1:a,3:c,1:b,3:b"
        // );
		printArray(comments);

		testSpeed(new CommentFrequency(false), comments);
		
	}

	private static void testSpeed(ICommentContainer cc, UserComment[] comments) {
		PrintUtil.println("======START");
		long begin = System.nanoTime();
		cc.setBeginTime(begin);
        for (int i = 0; i < comments.length; i++) {
            cc.insert(comments[i]);
        }
        // for (UserComment userComment : comments) {
        // cc.insert(userComment);
        // }
		PrintUtil.println("======END, cost:" + (System.nanoTime() - begin) + "\n");
	}

	private static void printArray(UserComment[] comments) {
		StringBuilder sb = new StringBuilder();
		for (UserComment comment : comments) {
			sb.append(comment).append(",");
		}
		sb.deleteCharAt(sb.length() - 1);
		PrintUtil.println(sb.toString());
	}

	private static UserComment[] getRandomStringArray(int size, int scope, int wordWidth, int userCount) {
		UserComment[] array = new UserComment[size];
		Random random = new Random(System.currentTimeMillis());
		for (int i = 0; i < array.length; i++) {
			long userId = System.nanoTime() % userCount + 1;
			StringBuilder sb = new StringBuilder();
			for (int j = 0; j < wordWidth; j++) {
				sb.append((char) ('a' + random.nextInt(scope)));
			}
			array[i] = new UserComment(userId, sb.toString());
		}
		return array;
	}

	private static UserComment[] parseFrom(String str) {
		String[] array = str.split(",");
		UserComment[] comments = new UserComment[array.length];
		for (int i = 0; i < comments.length; i++) {
			String[] pair = array[i].split(":");
			comments[i] = new UserComment(Long.parseLong(pair[0]), pair[1]);
		}
		return comments;
	}

}
```