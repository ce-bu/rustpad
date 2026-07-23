# Custom Container

This example demonstrates how to implement a container that holds a value of type `T` and drops it when the container is dropped. The container uses a raw pointer to hold the value, and a `PhantomData` marker to indicate that it owns a value of type `T`. The `Drop` implementation for the container uses the `#[may_dangle]` attribute to indicate that it may drop a value of type `T` without accessing it.

```
use std::fmt::Debug;

pub struct Container<T> {
    ptr: *const T,
    _marker: std::marker::PhantomData<T>,
}

impl<T> Container<T> {
    pub fn new(value: T) -> Self {
        let boxed = Box::new(value);
        let ptr = Box::into_raw(boxed);
        Container {
            ptr,
            _marker: std::marker::PhantomData,
        }
    }
}

unsafe impl<#[may_dangle] T> Drop for Container<T> {
    fn drop(&mut self) {
        println!("~Container");
        unsafe {
            Box::from_raw(self.ptr.cast_mut());
        }
    }
}

impl<T> std::ops::Deref for Container<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe { &*self.ptr }
    }
}

impl<T> std::ops::DerefMut for Container<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe { &mut *self.ptr.cast_mut() }
    }
}

struct Inspector<T: Debug>(T);

impl<T> Drop for Inspector<T>
where
    T: Debug,
{
    fn drop(&mut self) {
        println!("~Inspector: {:?}", self.0);
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_container() {
        // This code does not compile without eypatch.
        // Because Container implements Drop compiler will assume that it may access the value of type T during Drop,
        // and therefore assumes Container holds a mutable reference to y for the whole lifetime of c, which is not true.
        // In case of Box the compiler knows that Box does not access the value during Drop,
        // so it does not assume that Box holds a mutable reference to y for the whole lifetime of c.
        //
        // let mut y = 42;
        // let c = Container::new(&mut y);
        // println!("{}", y);

        // This code compiles without PhantomData, but it is not safe, regardless of the factt that container implements Drop or not.
        // The Inspector holds a refernce to z and we are accessing z while the Container is live and holds indirectly a mutable reference to it.
        // Container does not access z during Drop, we allready said that using `may_dangle` on the Drop impl.
        // However it drops the Inspector and the Inspector does access z during Drop, so this is not safe.
        //
        // We need to tell the compiler that we will drop the Inspector (T).
        // If Inspector does not implement Drop the code compiles with PhantomData.
        // The takeaway is that Drop is recursive.
        //
        // MIRI reports this as a retagged pointer error, which means that the pointer to z has been retagged as a mutable reference,
        // which is not allowed while it is still being accessed.
        //
        // let mut z = 42;
        // let c = Container::new(Inspector(&mut z));
        // println!("z={}", z);
    }

    fn check_box_variance<'a, 'b, T>(c: &'a Box<&'a T>) -> &'b Box<&'b T>
    where
        'a: 'b,
    {
        c
    }

    fn check_container_variance<'a, 'b, T>(c: &'a Container<&'a T>) -> &'b Container<&'b T>
    where
        'a: 'b,
    {
        c
    }
    #[test]
    fn test_variance() {}
}
```
