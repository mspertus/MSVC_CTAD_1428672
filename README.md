## Comments on MSVC Alias Template Deduction Issue 
I believe Bryce’s example in the linked [MSVC Developer Community Issue on mdspan CTAD](https://developercommunity.visualstudio.com/t/Alias-template-argument-deduction-leads/1428672) should work as he intended.

To avoid ambiguity, I’ve slightly modified Bryce’ godbolt to use `EltType` for the alias and `ElementType` for the guide ([https://godbolt.org/z/aM93PEWcz](https://godbolt.org/z/aM93PEWcz))

The deduction guide `f` of `basic_mdspan` is 
```c++
template <class ElementType, class... IndexType>
explicit basic_mdspan(ElementType*, IndexType...)
  -> basic_mdspan<ElementType, extents<__make_dynamic_extent<IndexType>()...>>;
```

According to over.match.class.deduct/2, we deduce the template arguments of `basic_mdspan<ElementType, extents<__make_dynamic_extent<IndexType>()...>>` from `basic_mdspan<EltType, extents<Extents...>>;` to deduce `EltType` for `ElementType` while `IndexType` is in a non-deducible context and therefore not deduced.

Again, according to over.match.class.deduct/2, we substitute these deductions into `f` to get a `g` with function type   
```c++
basic_mdspan(EltType*, IndexType...)
  -> basic_mdspan<EltType, extents<__make_dynamic_extent<IndexType>()...>>;
```
 Now we form a guide `f'` as follows.

*   over.match.class.deduct/2.1: The function type of `f'` is the function type of `g` shown above
*   over.match.class.deduct/2.2: The template parameters of `f'` are the template parameters of the `mdspan` that appear in the deduction (i.e., `EltType`) and the template parameters of `f` that were not deduced (`IndexType` was the only template parameter of the guide that was not deduced)
*   over.match.class.deduct/2.3: There is an `is_mdspan` constraint that the template arguments of `mdspan` are deducible from the return type
*   over.match.class.deduct/2.6: `f'` is `explicit` because `f` is.

Putting these together gives the following guide   
```c++
template <class EltType, class... IndexType>
explicit basic_mdspan(EltType*, IndexType...)
requires is_mdspan<basic_mdspan<EltType, extents<__make_dynamic_extent<IndexType>()...>>>
  -> basic_mdspan<EltType, extents<__make_dynamic_extent<IndexType>()...>>;
```

 Now line 49 of the godbolt should successfully deduce `double` for `EltType` and `int, int` for `IndexType...`, which clearly satisfies the constraint.

Please let me know if I’ve missed anything,

Mike   

