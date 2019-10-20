# Episode 1: Windows programming in Rust

This is the first episode in our journey to explore each library function we use in our code bases.

Below is the minimal C++ code to draw a window on screen. This code is from the Microsoft [MSDN webpage](https://docs.microsoft.com/en-us/windows/win32/learnwin32/your-first-windows-program). The goal is to translate this code into the [Rust programming language](https://www.rust-lang.org/), while analyzing the implementation of every (std) library function that we encounter along the way.
A secondary goal is that we will learn a lot of windows programming as well.  

```C++
#ifndef UNICODE
#define UNICODE
#endif 

#include <windows.h>

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam);

int WINAPI wWinMain(HINSTANCE hInstance, HINSTANCE, PWSTR pCmdLine, int nCmdShow)
{
    // Register the window class.
    const wchar_t CLASS_NAME[]  = L"Sample Window Class";
    
    WNDCLASS wc = { };

    wc.lpfnWndProc   = WindowProc;
    wc.hInstance     = hInstance;
    wc.lpszClassName = CLASS_NAME;

    RegisterClass(&wc);

    // Create the window.

    HWND hwnd = CreateWindowEx(
        0,                              // Optional window styles.
        CLASS_NAME,                     // Window class
        L"Learn to Program Windows",    // Window text
        WS_OVERLAPPEDWINDOW,            // Window style

        // Size and position
        CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT, CW_USEDEFAULT,

        NULL,       // Parent window    
        NULL,       // Menu
        hInstance,  // Instance handle
        NULL        // Additional application data
        );

    if (hwnd == NULL)
    {
        return 0;
    }

    ShowWindow(hwnd, nCmdShow);

    // Run the message loop.

    MSG msg = { };
    while (GetMessage(&msg, NULL, 0, 0))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    return 0;
}

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
        case WM_DESTROY:
            PostQuitMessage(0);
            return 0;

        case WM_PAINT:
            {
                PAINTSTRUCT ps;
                HDC hdc = BeginPaint(hwnd, &ps);
                FillRect(hdc, &ps.rcPaint, (HBRUSH) (COLOR_WINDOW+1));
                EndPaint(hwnd, &ps);
            }
        return 0;
    }
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}
```

First we will have to deal with how Windows works with Strings. 

>Windows represents Unicode characers using UTF-16 encoding, in which each character is encoded as a 16-bit value. UTF-16 characters are called _wide_ characters to distinguish them from 8-bit ANSI characters. The visual C++ compiler supports the built-in data type **wchar_t** for wide characters.

This is different from strings (both [String](https://doc.rust-lang.org/std/string/struct.String.html) and [&str](https://doc.rust-lang.org/std/primitive.str.html) types) in Rust, which are UTF-8.

Therefore what we need is a way to convert our UTF-8 encoding to UTF-16. Looking at the std library we note that *&str* has exactly such a method: [encode_utf16()](https://doc.rust-lang.org/std/string/struct.String.html#method.encode_utf16). This functions returns an iterator of ```u16``` over the string encoded as UTF-16. In our use case we would write something along the lines of:

```rust
    /// from the 'winit' crate.
    pub fn to_wide(&self, text: &str) {
        let text = OsStr::new(text)
            .encode_wide()
            .chain(Some(0).into_iter())
            .collect::<Vec<_>>();
    }

    /// from druid crate
    fn to_wide_sized(&self) -> Vec<u16> {
        self.as_ref().encode_wide().collect()
    }
    fn to_wide(&self) -> Vec<u16> {
        self.as_ref().encode_wide().chain(Some(0)).collect()
    }

    /// Convert a Rust UTF-8 `string` into a NUL-terminated UTF-16 vector
    fn str_to_utf16(string: &str) -> Vec<u16> {
        let mut ret: Vec<u16> = string
            .encode_utf16()
            .chain(Some(0))
            .collect();
        ret
    }
```
Here's the method implemenmtation of ```encode_utf16```:
```rust
#[stable(feature = "encode_utf16", since = "1.8.0")]
pub fn encode_utf16(&self) -> EncodeUtf16<'_> {
    EncodeUtf16 { chars: self.chars(), extra: 0 }
}
```

The iterator is a struct called ```EncodeUtf16```: 

```rust
#[derive(Clone)]
#[stable(feature = "encode_utf16", since = "1.8.0")]
pub struct EncodeUtf16<'a> {
    chars: Chars<'a>,
    extra: u16,
}
```

Lets have a look at [Chars](https://doc.rust-lang.org/std/str/struct.Chars.html).
 > An iterator over the chars of a string slice. This struct is created by the [chars](https://doc.rust-lang.org/std/primitive.str.html#method.chars) method on str.

```rust
#[derive(Clone)]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct Chars<'a> {
    iter: slice::Iter<'a, u8>
}
```

The ```chars``` method implementation is as follows:

```rust
#[stable(feature = "rust1", since = "1.0.0")]
    #[inline]
    pub fn chars(&self) -> Chars<'_> {
        Chars{iter: self.as_bytes().iter()}
    }
```



The ```as_bytes()``` str method converts a string slice to a byte slice.

```rust
#[stable(feature = "rust1", since = "1.0.0")]
    #[inline(always)]
    #[rustc_const_unstable(feature="const_str_as_bytes")]
    pub const fn as_bytes(&self) -> &[u8] {
        union Slices<'a> {
            str: &'a str,
            slice: &'a [u8],
        }
        unsafe { Slices { str: self }.slice }
    }
```

Wow, that's our first encounter with nifty code! We will have to look into the union concept later.

The primitive [slice](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.iter) type of an 8-bit unsigned integer type [u8](https://doc.rust-lang.org/std/primitive.u8.html) provides a
> dynamically-sized view into a contiguous sequence, [T].

The [iter()](https://doc.rust-lang.org/stable/std/slice/struct.Iter.html)  method for ```slice``` returns an Iter\<T>, which is an immutable iterator over the slice.

```rust
#[stable(feature = "rust1", since = "1.0.0")]
    #[inline]
    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            let ptr = self.as_ptr();
            assume(!ptr.is_null());

            let end = if mem::size_of::<T>() == 0 {
                (ptr as *const u8).wrapping_add(self.len()) as *const T
            } else {
                ptr.add(self.len())
            };

            Iter {
                ptr,
                end,
                _marker: marker::PhantomData
            }
        }
    }
```
and 

```rust
#[stable(feature = "rust1", since = "1.0.0")]
pub struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T, // If T is a ZST, this is actually ptr+len.  This encoding is picked so that
                   // ptr == end is a quick test for the Iterator being empty, that works
                   // for both ZST and non-ZST.
    _marker: marker::PhantomData<&'a T>,
}
```

We are starting to get deep down the Rust here!

the Iter struct implements the [std::iter::Iterator](https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html) trait. One of the required methods for this trait is [collect](https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html#method.collect). ```collect``` transforms an iterator into  collection:
> The most basic pattern in which collect() is used is to turn one collection into another. You take a collection, call iter on it, do a bunch of transformations, and then collect() at the end.

The collect method implementation is as follows: 
```rust
#[inline]
    #[stable(feature = "rust1", since = "1.0.0")]
    #[must_use = "if you really need to exhaust the iterator, consider `.for_each(drop)` instead"]
    fn collect<B: FromIterator<Self::Item>>(self) -> B where Self: Sized {
        FromIterator::from_iter(self)
    }
```
> By implementing FromIterator for a type, you define how it will be created from an iterator. This is common for types which describe a collection of some kind.
FromIterator's from_iter is rarely called explicitly, and is instead used through Iterator's collect method. See collect's documentation for more examples.

The [from_iter](https://doc.rust-lang.org/stable/std/iter/trait.FromIterator.html#tymethod.from_iter) method of this trait  returns a value from an iterator.

A from_iter implememntation for String looks like this: 
```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl FromIterator<char> for String {
    fn from_iter<I: IntoIterator<Item = char>>(iter: I) -> String {
        let mut buf = String::new();
        buf.extend(iter);
        buf
    }
}
```

and for Vec, like this:
```Rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T> FromIterator<T> for Vec<T> {
    #[inline]
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Vec<T> {
        <Self as SpecExtend<T, I::IntoIter>>::from_iter(iter.into_iter())
    }
}
```

