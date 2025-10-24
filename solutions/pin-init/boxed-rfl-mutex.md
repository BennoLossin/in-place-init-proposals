```rust
#[repr(transparent)]
pub struct Opaque<T> {
    value: UnsafeCell<MaybeUninit<T>>,
    _pin: PhantomPinned, // should be replaced with `UnsafePinned`
}

impl<T> Opaque<T> {
    pub fn ffi_init(init_func: FnOnce(*mut T)) -> impl PinInit<Self>;
}
```

```rust
mod bindings {
    pub type mutex;

    pub unsafe fn __mutex_init(ptr: *mut mutex);
}
```

```rust
#[pin_data]
pub struct Mutex<T> {
    #[pin]
    mutex: Opaque<bindings::mutex>,
    #[pin]
    value: UnsafeCell<T>,
}

impl<T> Mutex<T> {
    pub fn new<E>(value: impl PinInit<T, E>) -> impl PinInit<Self, E>
    where
        E: From<Infallible>,
    {
        pin_init!(Self {
            mutex <- Opaque::ffi_init(|ptr| unsafe { bindings::__mutex_init(ptr) }),
            value <- UnsafeCell::pin_init(value),
        })
    }
}
```

```rust
pub type DriverData;

impl DriverData {
    fn new() -> impl PinInit<DriverData, Error>;
}
```

```rust
fn problem() -> Result<Pin<Box<Mutex<DriverData>>>, Error> {
    Box::pin_init(Mutex::new(DriverData::new()))
}
```
