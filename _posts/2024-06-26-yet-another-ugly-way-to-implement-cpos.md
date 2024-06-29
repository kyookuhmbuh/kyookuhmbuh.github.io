---
title: Yet another ugly way to implement CPOs
description: Just a silly speculation about whether the shown experimental way is viable.
categories: [c++, experiments]
tags: [c++, templates, traits]
---

### Links

- [Source code](https://gist.github.com/kobtsev/47568600f274dcab5955f2527894a0ab) on GitHub Gist;
- [Some tests and playground](https://godbolt.org/z/oW8d9WKsP) on Compiler Explorer.
 
### What do we have now?

 We have several popular approaches to statically implement customization points:

- Free (hidden friend) functions found by ADL: `swap`, `begin`, `end`, etc;
- [Niebloid](https://brevzin.github.io/c++/2020/12/19/cpo-niebloid/) approach based [CPOs](https://en.cppreference.com/w/cpp/ranges/cpo) and [tag_invoke](https://wg21.link/p1895r0): `std::ranges`, `std::execution`;
- Class template specialization: `std::hash`, `std::formatter`, `std::is_error_code_enum`, etc.
        
### Do you believe in a bright future?

The following proposals may help improve your mental health a little:

- [P2279R0](https://wg21.link/p2279r0): We need a language mechanism for customization points;
- [P2547R1](https://wg21.link/p2547r1): Language support for customisable functions.

I can only wish that someday there will be language support for proper customization.

### Class template specialization way again

Let's start from the very beginning. We have some class template that describes a specific [trait](https://doc.rust-lang.org/book/ch10-02-traits.html) and is intended to be customized. My naming skills leave a lot to be desired, so let's just call this template `trait_impl`.

```c++
namespace extra {

  template <typename Target>
  struct trait_impl;

}
```

We could also check for specialization by writing something like this:

```c++
namespace extra {  

  template <typename T>
  concept has_trait = requires { trait_impl<T>{}; };

}
```

In order not to generate new instances when using customization (simply because it's inconvenient), we will make an auxiliary template:

```c++
namespace extra {  

  template <typename T> requires has_trait<T>
  inline constexpr auto trait = trait_impl<T>{};

}
```

How can we make a specialization for `trait_impl`? Let's do something like this:

```c++
namespace my_lib {

  struct nonsense {
    int value;
  };

} // namespace my_lib

namespace extra {

  template <>
  struct trait_impl<my_lib::nonsense> {
    constexpr auto operator()(my_lib::nonsense const& target) noexcept {
      return target.value;
    }
  }; 

} // namespace extra
```

Class template specialization is a fairly flexible way of customization that allows us to do a lot. 
Also allows us to create customizations not only for functional objects but also for complex interfaces like *[std::formatter](https://en.cppreference.com/w/cpp/utility/format/formatter)*.
But... But what annoys me personally about this is the difficult implementation of specializations.
Specialization must be in the library namespace, which in turn forces us to dance around that namespace: tearing apart client code.

### Silly tags again

What if we improve the approach a little by applying the idea of ​​using tags from the `tag_invoke` approach?
Let's upgrade the source code of the auxiliary library using tags:

```c++
namespace extra {

  template <typename Tag, typename T>
  struct trait_impl;

  template <typename T, typename Tag>
  concept has_trait = requires { trait_impl<Tag, T>{}; };

  template <typename Tag, has_trait<Tag> T>
  inline constexpr auto trait = trait_impl<Tag, T>{};

}
```

The result was several templates with the following parameters:

- Tag: A tag that specifies which customization is used;
- T: A target type for which the customization is intended;

Now let's create a tag for a trait. To complicate the task, let's move it into a separate namespace:

```c++
namespace domain {
  struct get_value; // trait tag
} 
```
    
Let me remind you that we have a template designed for creating specializations:

```c++
namespace extra {

  template <typename Tag, typename T>
  struct trait_impl;

} 
```

Let's "infect" the target class with a nested template class. Invade it like a virus and do unattractive things.

```c++
namespace client {

  struct target {
    int value;

    template <typename>
    struct trait_impl; // inject name
  };

} 
```

Now we can implement new traits for the target class using tags:

```c++
namespace client {

  struct target 
  {
    int value;

    // inject name 
    template <typename> 
    struct trait_impl; 

    // inline trait impl (T::trait_impl<Tag>)
    template <>
    struct trait_impl<domain::some_trait_1> { /* ... */ }; 
  };

  // impl get_value for target
  template <>
  struct target::trait_impl<domain::get_value>  
  {
    constexpr auto operator()(target const& t) const noexcept 
    {
      return t.value;
    }
  }; 

  // impl some_trait_2 for target
  template <>
  struct target::trait_impl<domain::some_trait_2> { /* ... */ }; 

} // namespace client 
```

Now let's teach the auxiliary library to look for trait implementations. 
Naming entities is quite a difficult job =)

```c++
namespace extra {

  // T::trait_impl<Tag>
  template <typename T, typename Tag>
  concept has_trait_impl_from_target_nested_type = requires {
    { typename T::template trait_impl<Tag>{} };
  };

  template <typename Tag, typename T>
  struct trait_impl;

  template <typename Tag, typename T>
    requires has_trait_impl_from_target_nested_type<T, Tag>
  struct trait_impl<T, Tag> : T::template trait_impl<Tag> {};

} // namespace extra
```

Maybe this will work:

```c++
int main([[maybe_unused]] int argc, [[maybe_unused]] char** argv)
{
  static_assert(extra::has_trait<client::target, domain::get_value>);

  struct ignorant{};
  static_assert(not extra::has_trait<ignorant, domain::get_value>);
    
  constexpr client::target instance { 12 };

  // Yep, call requires a lot of code =( 
  constexpr auto trait = extra::trait<domain::get_value, client::target>;
  constexpr auto value = trait(instance); 
    
  static_assert(12 == value);
}  
```

### Let's make tags more useful

You can turn a tag into an invocable: the tag will look and behave like a CPO.

```c++
namespace domain {
  
  inline constexpr struct final get_value_t 
  {

    // boilerplate-code with forward args, noexcept, trailing return type

    template <extra::has_trait<get_value_t> T>
    constexpr auto operator()(T const& target) const 
      noexcept(noexcept(extra::trait<get_value_t, T>(target))) 
    {
      return extra::trait<get_value_t, T>(target);
    } 

  } get_value;

} // namespace domain 
```

I don't do this because I don't see the need for it. It seems to me that such tricks lead to boilerplate code growth. 
Personally I'd rather live with the special specialization of `trait_impl` which is used to infer the target type from the first parameter of the `trait_impl<Tag, T>::operator()`: 
    
```c++
namespace extra {

  template <typename Tag, typename T = void>
  struct trait_impl;

  template <typename Tag, has_trait<Tag> T = void>
  inline constexpr auto trait = trait_impl<Tag, T>{};
  
  template <typename Tag>
  struct trait_impl<Tag, void>
  {
    using is_transparent = std::true_type;

    template <typename T,
              typename... U,
              typename Target = std::remove_cvref_t<T>>
        requires has_trait<Target, Tag> and
                 std::invocable<trait_impl<Tag, Target>, T, U...>
    inline constexpr auto operator()(T&& target, U&&... args) const
        noexcept(std::is_nothrow_invocable_v<trait_impl<Tag, Target>, T, U...>)
        -> std::invoke_result_t<trait_impl<Tag, Target>, T, U...>
    {
        return trait<Tag, Target>(std::forward<T>(target),
                                  std::forward<U>(args)...);
    }
  };

} // namespace extra

int main([[maybe_unused]] int argc, [[maybe_unused]] char** argv)
{
  using namespace client;
  using namespace domain;

  static_assert(extra::has_trait<target, get_value>);
  
  constexpr auto instance = target{ 12 };

  // now less code is required to call 
  constexpr auto value = extra::trait<get_value>(instance);
    
  static_assert(12 == value);
}  

```

It is more important when declaring a trait, if we are its owner, to provide a default implementation.
Let's try using tags for this:

```c++
namespace domain {
  
  struct get_value 
  {
    // inject name 
    template <typename> 
    struct trait_impl; 

    // default impl
    template <typename T>
    struct trait_impl<T> 
    {
      constexpr auto operator()(T const&) const noexcept 
      {
        return 0;
      } 
    }; 

    // delegating impl
    template <extra::has_trait<get_value> T>
    struct trait_impl<std::optional<T>> 
    {
      constexpr auto operator()(std::optional<T> const& opt) const noexcept 
      {
        if (opt) 
        {
          return extra::trait<get_value, T>(*opt);
        }

        return 0;
      } 
    }; 

  };

} // namespace domain 
```

This time we will teach the auxiliary library to extract specializations for a trait from a tag:

```c++
namespace extra {

  // Tag::trait_impl<T>
  template <typename T, typename Tag>
  concept has_trait_impl_from_tag_nested_type =
    not std::is_void_v<T> // you cannot create a reference to 'void'
    and requires { { typename Tag::template trait_impl<T>{} }; };

  template <typename Tag, typename T>
    requires has_trait_impl_from_tag_nested_type<T, Tag> and
             (not has_trait_impl_from_target_nested_type<T, Tag>)
  struct trait_impl<Tag, T> : Tag::template trait_impl<T> {};

} // namespace extra
```

Note that we have made target type specialization preferable to tag specialization (thanks to the require-clause).

### What about enums and third party types?

Imagine that we are not the owner of the tag for the trait. 
This means we cannot provide a default implementation using a tag, as we already did. 
But we can still assume a specialization for a trait within its own namespace. 
Seems better than nothing. If we only have third party types and do not own the trait, most likely there is something wrong in the code architecture.

We also cannot put nested types inside enums. 
We can do some naughtiness with ADL to support enums in our schema.

```c++
namespace extra {
  
  namespace internal::trait_impl_from_adl
  {
    template <typename... U>
    void trait_impl(U...) = delete;

    template <typename... U>
    auto adl_helper(U... args)
    {
      return trait_impl(args...);
    }

    template <typename T, typename Tag>
    concept type_check = requires {
      {
        trait_impl(std::type_identity<Tag>{}, std::type_identity<T>{})
      } -> std::default_initializable;
    };

    struct type_getter
    {
      template <typename... U>
      constexpr auto operator()(U... args) noexcept
      {
        return adl_helper(args...);
      }
    };

    template <typename Tag, typename T>
    using type = std::invoke_result_t<type_getter,
                                      std::type_identity<Tag>,
                                      std::type_identity<T>>;
  } // namespace internal::trait_impl_from_adl

  template <typename T, typename Tag>
  concept has_trait_impl_from_adl =
    internal::trait_impl_from_adl::type_check<T, Tag>;

  template <typename Tag, typename T>
    requires has_trait_impl_from_adl<T, Tag>
  struct trait_impl<Tag, T> : internal::trait_impl_from_adl::type<Tag, T> {};

} // namespace extra
```

 Let's try to use them:
    
```c++
// some third party traits
namespace domain { 
  struct to_string; 
  struct validate; 
}

enum class my_kind { first, second };

struct my_kind_to_string {
  constexpr char const* operator()(my_kind e) const noexcept {
    switch (e) {
      case my_kind::first: return "first";
      case my_kind::second: return "second";
      default: std::unreachable();
    }
  }
};

// use ADL as deduction guide 
auto trait_impl(
    std::type_identity<domain::to_string>, 
    std::type_identity<my_kind>) 
  -> my_kind_to_string;

// or make trait type getter
inline constexpr auto trait_impl(
    std::type_identity<domain::validate>,
    std::type_identity<my_kind>) noexcept 
{
  return [](my_kind e) {
    switch (e) {
      case my_kind::first: 
      case my_kind::second: 
        return true;
      default: 
        return false;
    }
  };
}

int main([[maybe_unused]] int argc, [[maybe_unused]] char** argv) {
  constexpr my_kind e = my_kind::first;
  static_assert(extra::trait<domain::validate>(e));

  auto kind_str = extra::trait<domain::to_string>(e);
  // ...
}  
```

### Proper customization points

I took the formal criteria for proper customization points from the article "[Barry Revzin: Why tag_invoke is not the solution I want](https://brevzin.github.io/c++/2020/12/01/tag-invoke/)". 

1. The ability to see clearly, in code, what the interface is that can (or needs to) be customized.
2. The ability to provide default implementations that can be overridden, not just non-defaulted functions.
3. The ability to opt in explicitly to the interface.
4. The inability to incorrectly opt in to the interface (for instance, if the interface has a function that takes an int, you cannot opt in by accidentally taking an unsigned int).
5. The ability to easily invoke the customized implementation. Alternatively, the inability to accidentally invoke the base implementation.
6. The ability to easily verify that a type implements an interface.

The method I suggested (as a slight improvement on class template specialization way) **probably doesn't meet these criteria either**. 
Does the proposal [P2547R1](https://wg21.link/p2547r1) (Language support for customisable functions) meet these criteria?
How do you see the future mechanism for creating customization points built into the language?
Share what approaches you use to create customization points.                                        
