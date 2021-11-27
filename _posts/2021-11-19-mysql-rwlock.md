---
layout: post
title:  "mysql 读写锁解析"
categories: innodb
---

* content
{:toc}

## 背景
mysql有很多锁，今天看下读写锁是怎么实现的，5.7版本新增了sx锁，先从简单的没有sx锁的版本。<br/>

## 结构体
先来看下结构体，下面是精简版<br/>
```
struct rw_lock_t
{
	volatile lint	lock_word;  //计数器
	volatile ulint	waiters; // = 1表示有线程在等锁，可能等写锁，可能等读锁
	volatile bool	recursive; // 是否递归锁
	volatile os_thread_id_t	writer_thread; // 持有写锁的线程id
	os_event_t	event;  // 等待信号量
	os_event_t	wait_ex_event; // 扩展等待信号量，写锁排队的时候使用
	/** The mutex protecting rw_lock_t */
	mutable ib_mutex_t mutex; //锁保护，可以用原子操作代替
};
```
lock_word初始化是X_LOCK_DECR，每次读锁的时候，lock_word就减1，每次写锁的时候，lock_word就减X_LOCK_DECR。<br/>
下面是lock_word取值范围的说明：<br/>
lock_word == X_LOCK_DECR:      Unlocked. <br/>

0 < lock_word < X_LOCK_DECR: <br/> 
Read locked, no waiting writers. (X_LOCK_DECR - lock_word) is the number of readers that hold the lock. <br/>

lock_word == 0:		       Write locked <br/>

-X_LOCK_DECR < lock_word < 0: <br/> 
Read locked, with a waiting writer. (-lock_word) is the number of readers that hold the lock. <br/>

lock_word <= -X_LOCK_DECR: <br/>
Recursively write locked. lock_word has been decremented by X_LOCK_DECR once for each lock, so the number of locks is: ((-lock_word) / X_LOCK_DECR) + 1 <br/>

When lock_word <= -X_LOCK_DECR, we also know that lock_word % X_LOCK_DECR == 0: other values of lock_word are invalid. <br/>

![avatar](/images/rw_lock.jpg)

## 读锁
mysql里面的s锁就是读锁, 读锁调用的都是rw_lock_s_lock_func函数，下面是函数精简实现。
先调用rw_lock_s_lock_low函数，如果lock_word > 0, 就减1, 加锁成功。<br/>
否则就调用rw_lock_s_lock_spin函数等待(这样可以保证写锁不会饿死)。先判断lock_word是否 <= 0, 如果是的话，就先让出cpu <br/>
再次尝试调用rw_lock_s_lock_low，加锁成功返回，否则就申请cell, wait等待在lock->event。<br/>
如果被唤醒再次循环刚才的流程 <br/>
```
bool
rw_lock_lock_word_decr(
/*===================*/
	rw_lock_t*	lock,		/*!< in/out: rw-lock */
	ulint		amount,		/*!< in: amount to decrement */
	lint		threshold)	/*!< in: threshold of judgement */
{
	bool success = false;
	mutex_enter(&(lock->mutex));
	if (lock->lock_word > threshold) {
		lock->lock_word -= amount;
		success = true;
	}
	mutex_exit(&(lock->mutex));
	return(success);
}
@return TRUE if success */
bool rw_lock_s_lock_low(rw_lock_t*	lock)
{
	if (!rw_lock_lock_word_decr(lock, 1, 0)) {
		/* Locking did not succeed */
		return(false);
	}

	return(true);	/* locking succeeded */
}
void rw_lock_s_lock_func(
	rw_lock_t*	lock,	/*!< in: pointer to rw-lock */
	ulint		pass,	/*!< in: pass value; != 0, if the lock will
				be passed to another thread to unlock */
	const char*	file_name,/*!< in: file name where lock requested */
	ulint		line)	/*!< in: line where requested */
{
	if (!rw_lock_s_lock_low(lock, pass, file_name, line)) {
		/* Did not succeed, try spin wait */
		rw_lock_s_lock_spin(lock, pass, file_name, line);
	}
}

void rw_lock_s_lock_spin(
	rw_lock_t*	lock,	/*!< in: pointer to rw-lock */
	ulint		pass,	/*!< in: pass value; != 0, if the lock
				will be passed to another thread to unlock */
	const char*	file_name, /*!< in: file name where lock requested */
	ulint		line)	/*!< in: line where requested */
{
	ulint		i = 0;	/* spin round count */
	sync_array_t*	sync_arr;
	ulint		spin_count = 0;
	uint64_t	count_os_wait = 0;

lock_loop:

	/* Spin waiting for the writer field to become free */
	os_rmb;
	while (i < srv_n_spin_wait_rounds && lock->lock_word <= 0) {
		if (srv_spin_wait_delay) {
			ut_delay(ut_rnd_interval(0, srv_spin_wait_delay));
		}

		i++;
	}

	if (i >= srv_n_spin_wait_rounds) {
		os_thread_yield();
	}

	++spin_count;

	/* We try once again to obtain the lock */
	if (rw_lock_s_lock_low(lock, pass, file_name, line)) {
		return; /* Success */
	} else {

		if (i < srv_n_spin_wait_rounds) {
			goto lock_loop;
		}


		++count_os_wait;

		sync_cell_t*	cell;

		sync_arr = sync_array_get_and_reserve_cell(
				lock, RW_LOCK_S, file_name, line, &cell);

		/* Set waiters before checking lock_word to ensure wake-up
		signal is sent. This may lead to some unnecessary signals. */
		rw_lock_set_waiter_flag(lock);

		if (rw_lock_s_lock_low(lock, pass, file_name, line)) {

			sync_array_free_cell(sync_arr, cell);
			return; /* Success */
		}

		sync_array_wait_event(sync_arr, cell);

		i = 0;

		goto lock_loop;
	}
}
```
在来看下解锁读锁，比较简单，就是对lock_word+1，如果lock_word== 0说明已经没有其他线程持有读锁，
并且有写锁等待就唤醒写锁 <br/>
```
void rw_lock_s_unlock_func(
	rw_lock_t*	lock)	/*!< in/out: rw-lock */
{

	/* Increment lock_word to indicate 1 less reader */
	lint	lock_word = rw_lock_lock_word_incr(lock, 1);
	if (lock_word == 0) {

		/* wait_ex waiter exists. It may not be asleep, but we signal
		anyway. We do not wake other waiters, because they can't
		exist without wait_ex waiter and wait_ex waiter goes first.*/
		os_event_set(lock->wait_ex_event);

	}
}
lint rw_lock_lock_word_incr(
	rw_lock_t*	lock,		/*!< in/out: rw-lock */
	ulint		amount)		/*!< in: amount of increment */
{
	lint local_lock_word;

	mutex_enter(&(lock->mutex));

	lock->lock_word += amount;
	local_lock_word = lock->lock_word;

	mutex_exit(&(lock->mutex));

	return(local_lock_word);
}

```

## 写锁
mysql里面的x锁就是写锁，底层调用的是rw_lock_x_lock_func函数。 <br/>
先调用rw_lock_x_lock_low函数，加锁成功就返回true，否则返回false。 <br/>
加锁失败的话，就申请一个cell，然后等待lock->event通知。<br/>
```
void rw_lock_x_lock_func(
	rw_lock_t*	lock,	/*!< in: pointer to rw-lock */
	ulint		pass,	/*!< in: pass value; != 0, if the lock will
				be passed to another thread to unlock */
	const char*	file_name,/*!< in: file name where lock requested */
	ulint		line)	/*!< in: line where requested */
{
	ulint		i = 0;
	sync_array_t*	sync_arr;
	ulint		spin_count = 0;
	uint64_t	count_os_wait = 0;

lock_loop:

	if (rw_lock_x_lock_low(lock, pass, file_name, line)) {
		/* Locking succeeded */
		return;

	} else {
		/* Spin waiting for the lock_word to become free */
		os_rmb;
		while (i < srv_n_spin_wait_rounds
		       && lock->lock_word <= 0) {

			if (srv_spin_wait_delay) {
				ut_delay(ut_rnd_interval(
						0, srv_spin_wait_delay));
			}

			i++;
		}

		spin_count += i;

		if (i >= srv_n_spin_wait_rounds) {

			os_thread_yield();

		} else {

			goto lock_loop;
		}
	}

	sync_cell_t*	cell;

	sync_arr = sync_array_get_and_reserve_cell(
			lock, RW_LOCK_X, file_name, line, &cell);

	/* Waiters must be set before checking lock_word, to ensure signal
	is sent. This could lead to a few unnecessary wake-up signals. */
	rw_lock_set_waiter_flag(lock);

	if (rw_lock_x_lock_low(lock, pass, file_name, line)) {
		sync_array_free_cell(sync_arr, cell);
		/* Locking succeeded */
		return;
	}

	sync_array_wait_event(sync_arr, cell);

	i = 0;

	goto lock_loop;
}
```
先看下rw_lock_x_lock_wait_func函数的实现。<br/>
如果lock_word >= threshold，直接return。<br/>
否则就申请cell，等待lock->wait_ex_event。 <br/>
```
void
rw_lock_x_lock_wait_func(
/*=====================*/
	rw_lock_t*	lock,	/*!< in: pointer to rw-lock */
	lint		threshold,/*!< in: threshold to wait for */
	const char*	file_name,/*!< in: file name where lock requested */
	ulint		line)	/*!< in: line where requested */
{
	ulint		i = 0;
	ulint		n_spins = 0;
	sync_array_t*	sync_arr;
	uint64_t	count_os_wait = 0;

	os_rmb;

	while (lock->lock_word < threshold) {

		if (srv_spin_wait_delay) {
			ut_delay(ut_rnd_interval(0, srv_spin_wait_delay));
		}

		if (i < srv_n_spin_wait_rounds) {
			i++;
			os_rmb;
			continue;
		}

		/* If there is still a reader, then go to sleep.*/
		++n_spins;

		sync_cell_t*	cell;

		sync_arr = sync_array_get_and_reserve_cell(
			lock, RW_LOCK_X_WAIT, file_name, line, &cell);

		i = 0;

		/* Check lock_word to ensure wake-up isn't missed.*/
		if (lock->lock_word < threshold) {
			sync_array_wait_event(sync_arr, cell);

			/* It is possible to wake when lock_word < 0.
			We must pass the while-loop check to proceed.*/

		} else {
			sync_array_free_cell(sync_arr, cell);
			break;
		}
	}
}
```
下面看具体的rw_lock_x_lock_low的实现。<br/>
首先比较lock_word，如果lock_word > 0，就减去X_LOCK_DECR，然后调用rw_lock_x_lock_wait_func函数，等待lock_word >= 0。条件满足，等待结束就表示加锁成功了。这种情况属于加写锁之前都是读锁，没有写锁<br/>
如果lock_word <= 0，说明之前已经有其他线程抢到写锁了。<br/>
如果之前加锁的线程不是本线程，那么加写锁失败，return false。<br/>
否则就是同一个线程再次加写锁，并且是递归锁的话，就减去X_LOCK_DECR。<br/>
```
bool rw_lock_x_lock_low(
/*===============*/
	rw_lock_t*	lock,	/*!< in: pointer to rw-lock */
	ulint		pass,	/*!< in: pass value; != 0, if the lock will
				be passed to another thread to unlock */
	const char*	file_name,/*!< in: file name where lock requested */
	ulint		line)	/*!< in: line where requested */
{
	if (rw_lock_lock_word_decr(lock, X_LOCK_DECR, 0)) {
		/* Decrement occurred: we are writer or next-writer. */
		rw_lock_set_writer_id_and_recursion_flag(lock, !pass);

		rw_lock_x_lock_wait(lock, pass, 0, file_name, line);

	} else {
		os_thread_id_t	thread_id = os_thread_get_curr_id();
                /* Decrement failed: relock or failed lock */
		if (!pass && lock->recursive
		    && os_thread_eq(lock->writer_thread, thread_id)) {
			/* Relock */
                        lock->lock_word -= X_LOCK_DECR;
		} else {
			/* Another thread locked before us */
			return(false);
		}
	}

	return(true);
}
```
下面再来看下写锁的解锁实现。<br/>
首先判断如果lock_word == 0说明是第一次加的写锁，那么lock_word + X_LOCK_DECR，然后判断是否有waiters(可能是读锁或者写锁等待)，如果有的话就通知一下其他waiters。<br/>
```
void rw_lock_x_unlock_func(
	rw_lock_t*	lock)	/*!< in/out: rw-lock */
{
	if (lock->lock_word == 0) {
		/* Last caller in a possible recursive chain. */
		lock->recursive = FALSE;
	}

	if (rw_lock_lock_word_incr(lock, X_LOCK_DECR) == X_LOCK_DECR) {
		/* There is 1 x-lock */
		/* atomic increment is needed, because it is last */

		if (lock->waiters) {
			rw_lock_reset_waiter_flag(lock);
			os_event_set(lock->event);
		}
        }
}
```

## sx锁
mysql新加的共享排它锁，先来看下相容性矩阵。<br/>
```
  | S|SX| X|
--+--+--+--+
S | o| o| x|
--+--+--+--+
SX| o| x| x|
--+--+--+--+
X | x| x| x|
--+--+--+--+
```
S锁和X锁与之前的逻辑相同，没有做变动，SX与SX和X互斥，与S共享，在加上SX锁之后，不会影响读操作，但阻塞写操作。 <br/>
背景参考 [内核文章](http://mysql.taobao.org/monthly/2015/07/05/) <br/>
lock_word新取值范围的说明：<br/>
lock_word == X_LOCK_DECR:	Unlocked. <br/>

X_LOCK_HALF_DECR < lock_word < X_LOCK_DECR: <br/>
S locked, no waiting writers.(X_LOCK_DECR - lock_word) is the number of S locks. <br/>

lock_word == X_LOCK_HALF_DECR:	SX locked, no waiting writers. <br/>
0 < lock_word < X_LOCK_HALF_DECR: <br/>

SX locked AND S locked, no waiting writers.(X_LOCK_HALF_DECR - lock_word) is the number of S locks. <br/>

lock_word == 0:			X locked, no waiting writers. <br/>

-X_LOCK_HALF_DECR < lock_word < 0: <br/>
S locked, with a waiting writer.(-lock_word) is the number of S locks. <br/>

lock_word == -X_LOCK_HALF_DECR:	X locked and SX locked, no waiting writers. <br/>

-X_LOCK_DECR < lock_word < -X_LOCK_HALF_DECR: <br/>
S locked, with a waiting writer which has SX lock. -(lock_word + X_LOCK_HALF_DECR) is the number of S locks. <br/>

lock_word == -X_LOCK_DECR:	X locked with recursive X lock (2 X locks). <br/>

-(X_LOCK_DECR + X_LOCK_HALF_DECR) < lock_word < -X_LOCK_DECR: <br/>
X locked. The number of the X locks is: 2 - (lock_word + X_LOCK_DECR) <br/>

lock_word == -(X_LOCK_DECR + X_LOCK_HALF_DECR): <br/>
X locked with recursive X lock (2 X locks) and SX locked. <br/>

lock_word < -(X_LOCK_DECR + X_LOCK_HALF_DECR): <br/>
X locked and SX locked.The number of the X locks is:2 - (lock_word + X_LOCK_DECR + X_LOCK_HALF_DECR) <br/>

####读锁
读锁的逻辑和原来没有变化，当lock_word > 0的时候可以加读写，成功以后lock_work - 1. <br/>
解锁的时候lock_work + 1, 多了一个变化是如果lock_work == -X_LOCK_HALF_DECR 唤醒wait_ex_event. <br/>

####写锁
加锁失败的时候，需要等待lock_word > X_LOCK_HALF_DECR. SX锁和X锁互斥。 <br/>
rw_lock_x_lock_low判断lock_word > X_LOCK_HALF_DECR，才会等待lock_word > 0，表示加锁成功。<br/>

####SX加锁
加锁逻辑和X锁差不多，如果rw_lock_sx_lock_low加锁成功直接返回，否则就等待lock->event直到lock_word > X_LOCK_HALF_DECR. <br/>
```
void rw_lock_sx_lock_func(
    rw_lock_t *lock,       /*!< in: pointer to rw-lock */
    ulint pass,            /*!< in: pass value; != 0, if the lock will
                           be passed to another thread to unlock */
    const char *file_name, /*!< in: file name where lock requested */
    ulint line)            /*!< in: line where requested */

{
  ulint i = 0;
  sync_array_t *sync_arr;


lock_loop:

  if (rw_lock_sx_lock_low(lock, pass, file_name, line)) {
    /* Locking succeeded */
    return;

  } else {

    /* Spin waiting for the lock_word to become free */
    os_rmb;
    while (i < srv_n_spin_wait_rounds && lock->lock_word <= X_LOCK_HALF_DECR) {
      if (srv_spin_wait_delay) {
        ut_delay(ut_rnd_interval(0, srv_spin_wait_delay));
      }

      i++;
    }

    if (i >= srv_n_spin_wait_rounds) {
      std::this_thread::yield();

    } else {
      goto lock_loop;
    }
  }

  sync_cell_t *cell;

  sync_arr =
      sync_array_get_and_reserve_cell(lock, RW_LOCK_SX, file_name, line, &cell);

  /* Waiters must be set before checking lock_word, to ensure signal
  is sent. This could lead to a few unnecessary wake-up signals. */
  rw_lock_set_waiter_flag(lock);

  if (rw_lock_sx_lock_low(lock, pass, file_name, line)) {
    sync_array_free_cell(sync_arr, cell);

    /* Locking succeeded */
    return;
  }

  ++count_os_wait;

  sync_array_wait_event(sync_arr, cell);

  i = 0;

  goto lock_loop;
}
```
rw_lock_sx_lock_low 先判断lock_word > X_LOCK_HALF_DECR, 如果成立就减去X_LOCK_HALF_DECR。<br/>
否则就判断之前加锁的线程是不是本线程，如果不是说明有其他线程加锁sx锁或者x锁，返回false,如果是就减去X_LOCK_HALF_DECR, 加锁成功。<br/>
```
bool rw_lock_sx_lock_low(
    rw_lock_t *lock,       /*!< in: pointer to rw-lock */
    ulint pass,            /*!< in: pass value; != 0, if the lock will
                           be passed to another thread to unlock */
    const char *file_name, /*!< in: file name where lock requested */
    ulint line)            /*!< in: line where requested */
{
  if (rw_lock_lock_word_decr(lock, X_LOCK_HALF_DECR, X_LOCK_HALF_DECR)) {

    /* Decrement occurred: we are the SX lock owner. */
    rw_lock_set_writer_id_and_recursion_flag(lock, !pass);

    lock->sx_recursive = 1;

  } else {
    /* Decrement failed: It already has an X or SX lock by this
    thread or another thread. If it is this thread, relock,
    else fail. */
    if (!pass && lock->recursive.load(std::memory_order_acquire) &&
        lock->writer_thread.load(std::memory_order_relaxed) ==
            std::this_thread::get_id()) {
      /* This thread owns an X or SX lock */
      if (lock->sx_recursive++ == 0) {
        lock->lock_word -= X_LOCK_HALF_DECR;
      }
    } else {
      /* Another thread locked before us */
      return false;
    }
  }
  return true;
}
```
####SX解锁
如果sx_recursive = 0表示sx锁都释放了，lock_word + X_LOCK_HALF_DECR。如果lock_word > X_LOCK_HALF_DECR 并且waiters = 1说明有写锁等待，通知一下。
```
static inline void rw_lock_sx_unlock_func(
    rw_lock_t *lock) /*!< in/out: rw-lock */
{
  --lock->sx_recursive;
  if (lock->sx_recursive == 0) {
    /* Last caller in a possible recursive chain. */
    if (lock->lock_word > 0) {
      if (rw_lock_lock_word_incr(lock, X_LOCK_HALF_DECR) <= X_LOCK_HALF_DECR) {
        ut_error;
      }
      /* Lock is now free. May have to signal read/write
      waiters. We do not need to signal wait_ex waiters,
      since they cannot exist when there is an sx-lock
      holder. */
      if (lock->waiters) {
        rw_lock_reset_waiter_flag(lock);
        os_event_set(lock->event);
        sync_array_object_signalled();
      }
    } else {
      /* still has x-lock */
      ut_ad(lock->lock_word == -X_LOCK_HALF_DECR ||
            lock->lock_word <= -(X_LOCK_DECR + X_LOCK_HALF_DECR));
      lock->lock_word += X_LOCK_HALF_DECR;
    }
  }
}
```

## 参考资料
http://mysql.taobao.org/monthly/2020/04/02/
