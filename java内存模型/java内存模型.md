**volatile**
	volatile是java虚拟机提供的轻量级同步机制，特点是1、可见性，2、禁止指令重拍。底层主要是通过内存屏障来实现的。
	不保证原子性。原子性需要用CAS。
**CAS 解决原子性**
	compare and swap。
	原子类atomicInteger。
	底层原理：自旋锁、unsafe。  
	unsafe类提供了硬件级别的院子操作。通过比较内存位置和预期原值，来决定是否更新值。
CAS源码分析

/**
 * Atomically exchanges the given value with the current value of
 * a field or array element within the given object <code>o</code>
 * at the given <code>offset</code>.
 *
 * @param o object/array to update the field/element in
 * @param offset field/element offset
 * @param newValue new value
 * @return the previous value
 * @since 1.8
 */
public final int getAndSetInt(Object o, long offset, int newValue) {
	int v;
	do {
		v = getIntVolatile(o, offset);
	} while (!compareAndSwapInt(o, offset, v, newValue));
	return v;
}

缺点：1、循环开销大；2、只能保证一个共享变量的原子操作；3、ABA问题？

ABA问题？原子更新引用？