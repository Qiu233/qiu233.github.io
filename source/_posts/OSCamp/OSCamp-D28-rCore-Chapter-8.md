---
title: OSCamp D28-rCore Chapter 8
date: 2024-05-05 19:53:57
tags: [Rust, RISC-V, Sync]
category: [OSCamp]
---
第八章的内容是同步，分为两个方面：
* 临界区域(Critical Section)：只允许一个或少数几个线程访问的代码块。
* 锁：限制访问的工具，有三种：

1. Mutex：互斥锁，只允许获取一次，其他获取必须等待上一个获取者释放。
2. Semaphore：信号量，相当于最多允许有限次获取的互斥锁，超出这个数量则需等待。
3. CondVar：条件变量，和锁一起使用，用于等待某个条件达成。

代码中实现的互斥锁有两种：
* MutexSpin：自旋锁，线程如果无法获取锁会直接回到调度队列等待下次调度。
* MutexBlocking：阻塞锁，线程如果无法获取锁会将自身状态设置为Blocking，直到锁的持有者释放时，`wakeup_task`会将等待队列中的线程唤醒并加入调度队列。

条件变量我的理解还不是很清楚，一句话描述大概是：
> 用同一个互斥锁在两个线程间实现**状态**的同步。

条件变量等待时会将之前获取的互斥锁释放，并阻塞自身。当某个线程将状态(例如某个变量)改变时，会调用signal函数将阻塞的线程唤醒，被唤醒的线程会立刻获得锁。

<!--more-->

# 死锁检测

rCore的死锁检测算法相当简单，这里放一个完全独立的实现(代码在文末)，因为并不是rCore内实现，所以应该不受荣誉准则限制。

需要注意的是几个矩阵的维护：

* 在调用lock或者down之前，Q加一。
* 在lock或者down成功后，Q减一、AVAIL减一、ALLOC加一。
* (可选)如果是动态检测死锁，应该在检测到之后选择一个死锁的线程干掉，此时必须释放掉它持有的锁，并且将ALLOC和AVAIL更新为释放后的状态。
* (课程要求)如果是申请时检测，应该：
    * 先将Q加一。
    * 调用detect检测在这种情况下是否会发生死锁，记录下来。
    * 将Q减一。
    * 如果前面的detect算法发现死锁，那么拒绝这次申请，返回`-0xDEAD`。
    * 注意这里整个流程都应该发生lock和down和对应的Q维护之前，因为是分配前的检查。

这个算法的缺点在于必须追踪每个线程分配的资源和数量，如果申请和释放的线程不同就会失效，例如测例里面的MPSC就过不了这种死锁检测。

## 代码实现

```Rust
use std::{convert::identity, sync::Mutex};

use lazy_static::lazy_static;


const NUM_SEM: usize = 2;
const NUM_THREADS: usize = 4;

lazy_static! {
    pub static ref Q: Mutex<Vec<Vec<usize>>> = Mutex::new(vec![vec![0; NUM_SEM]; NUM_THREADS]);
    pub static ref ALLOC: Mutex<Vec<Vec<usize>>> = Mutex::new(vec![vec![0; NUM_SEM]; NUM_THREADS]);
    pub static ref AVAIL: Mutex<Vec<usize>> = Mutex::new(vec![0; NUM_SEM]);
}


fn all_zero(v: &Vec<usize>) -> bool {
    v.iter().all(|x|*x == 0)
}

fn le_vec(v1: &Vec<usize>, v2: &Vec<usize>) -> bool {
    assert!(v1.len() == v2.len());
    v1.iter().zip(v2.iter()).all(|(x,y)|*x<=*y)
}

fn add_to(dst: &mut Vec<usize>, src: &Vec<usize>) {
    assert!(dst.len() == src.len());
    dst.iter_mut().zip(src.iter()).for_each(|(x,y)|*x += *y);
}

fn detect() {
    let q = Q.lock().unwrap();
    let alloc = ALLOC.lock().unwrap();
    let avail = AVAIL.lock().unwrap();
    let mut finish = vec![false; NUM_THREADS];
    finish.iter_mut()
        .enumerate()
        .filter(|(i, _)|all_zero(&alloc[*i]))
        .for_each(|(_, x)|*x = true);
    let mut W = avail.clone();
    loop {
        let mut found = None;
        for i in 0..NUM_THREADS {
            let cond1 = !finish[i];
            let cond2 = le_vec(&q[i], &W);
            if cond1 && cond2 {
                finish[i] = true;
                add_to(&mut W, &alloc[i]);
                found = Some(i);
                break;
            }
        }
        if found.is_none() {
            break;
        }
    }
    // !finish.into_iter().all(identity);
    for f in finish.into_iter().enumerate() {
        println!("finish[{}] = {}", f.0, f.1);
    }
}

fn main() {
    let q = vec![
        vec![1,0],
        vec![0,0],
        vec![0,1],
        vec![0,0],
    ];
    let alloc = vec![
        vec![0,1],
        vec![1,0],
        vec![1,0],
        vec![0,1],
    ];
    // let q = vec![
    //     vec![1,0,0],
    //     vec![0,0,1],
    //     vec![0,1,0],
    // ];
    // let alloc = vec![
    //     vec![0,1,0],
    //     vec![1,1,0],
    //     vec![0,0,1],
    // ];
    *Q.lock().unwrap() = q;
    *ALLOC.lock().unwrap() = alloc;
    let avai = vec![0,0];
    *AVAIL.lock().unwrap() = avai;
    detect();
}
```
