---
title: 'weak_ptr'
publishDate: '2025-1-03'
updatedDate: '2025-1-10'
description: 'C++中的weak_ptr'
tags:
  - C++
language: '中文'
draft: true
---

# weak_ptr真的不计数吗？是否有计数方式，在哪分配的内存？

**回答：弱引用也有引用计数**
`_Uses`表达的是当前有多少个强指针在引用内部指针，`_weaks`表达的是当前`_Ref_count`对象类型的使用次数，即如果`_weaks_count`计数的值为0，则会释放掉上面`_Ref_count`计数器对象。

1. `weak_ptr`和一般指针没有区别。
2. `weak_ptr`没有删除它能访问的动态分配的空间的能力（不修改`_User`）

它是对`shared_ptr`的一个访问者，不参与`shared_ptr`的引用计数，也不会删除所指向的对象。同时，当对象被`shared_ptr`删除后，`weak_ptr`就是无效的了。

使用`weak_ptr`，而不是使用`T*`这样的C++指针的原因是，`weak_ptr`是可以转换为`shared_ptr`参与引用计数，而`T*`不可以，（只有`new`出来的指针才可以构造`shared_ptr`）。也就是说，`weak_ptr`虽然不参与引用计数，但是引用计数机制是存在的，当需要转换为`shared_ptr`时就把这套机制传给`shared_ptr`。否则的话，使用`T*`构造`shared_ptr`的话，相当于有两套机制来管理引用计数，那么一个`shared_ptr`可能指向无效的对象（对象被另一套引用计数机制删除了）。

智能指针都继承自`_Ptr_base`，它会有两个成员：

~~~cpp
element_type* _Ptr;		//指向原始数据的指针
_Ref_count_base* _Rep;	//指向引用计数类的指针

//下面是完整代码

template <class _Ty>
class _Ptr_base { // base class for shared_ptr and weak_ptr
public:
    using element_type = remove_extent_t<_Ty>;

    _NODISCARD long use_count() const noexcept {
        return _Rep ? _Rep->_Use_count() : 0;
    }

    template <class _Ty2>
    _NODISCARD bool owner_before(const _Ptr_base<_Ty2>& _Right) const noexcept { // compare addresses of manager objects
        return _Rep < _Right._Rep;
    }

    _Ptr_base(const _Ptr_base&)            = delete;
    _Ptr_base& operator=(const _Ptr_base&) = delete;

protected:
    _NODISCARD element_type* get() const noexcept {
        return _Ptr;
    }

    constexpr _Ptr_base() noexcept = default;

    ~_Ptr_base() = default;

    template <class _Ty2>
    void _Move_construct_from(_Ptr_base<_Ty2>&& _Right) noexcept {
        // implement shared_ptr's (converting) move ctor and weak_ptr's move ctor
        _Ptr = _Right._Ptr;
        _Rep = _Right._Rep;

        _Right._Ptr = nullptr;
        _Right._Rep = nullptr;
    }

    template <class _Ty2>
    void _Copy_construct_from(const shared_ptr<_Ty2>& _Other) noexcept {
        // implement shared_ptr's (converting) copy ctor
        _Other._Incref();

        _Ptr = _Other._Ptr;
        _Rep = _Other._Rep;
    }

    template <class _Ty2>
    void _Alias_construct_from(const shared_ptr<_Ty2>& _Other, element_type* _Px) noexcept {
        // implement shared_ptr's aliasing ctor
        _Other._Incref();

        _Ptr = _Px;
        _Rep = _Other._Rep;
    }

    template <class _Ty2>
    void _Alias_move_construct_from(shared_ptr<_Ty2>&& _Other, element_type* _Px) noexcept {
        // implement shared_ptr's aliasing move ctor
        _Ptr = _Px;
        _Rep = _Other._Rep;

        _Other._Ptr = nullptr;
        _Other._Rep = nullptr;
    }

    template <class _Ty0>
    friend class weak_ptr; // specifically, weak_ptr::lock()

    template <class _Ty2>
    bool _Construct_from_weak(const weak_ptr<_Ty2>& _Other) noexcept {
        // implement shared_ptr's ctor from weak_ptr, and weak_ptr::lock()
        if (_Other._Rep && _Other._Rep->_Incref_nz()) {
            _Ptr = _Other._Ptr;
            _Rep = _Other._Rep;
            return true;
        }

        return false;
    }

    void _Incref() const noexcept {
        if (_Rep) {
            _Rep->_Incref();
        }
    }

    void _Decref() noexcept { // decrement reference count
        if (_Rep) {
            _Rep->_Decref();
        }
    }

    void _Swap(_Ptr_base& _Right) noexcept { // swap pointers
        _STD swap(_Ptr, _Right._Ptr);
        _STD swap(_Rep, _Right._Rep);
    }

    template <class _Ty2>
    void _Weakly_construct_from(const _Ptr_base<_Ty2>& _Other) noexcept { // implement weak_ptr's ctors
        if (_Other._Rep) {
            _Ptr = _Other._Ptr;
            _Rep = _Other._Rep;
            _Rep->_Incwref();
        } else {
            _STL_INTERNAL_CHECK(!_Ptr && !_Rep);
        }
    }

    template <class _Ty2>
    void _Weakly_convert_lvalue_avoiding_expired_conversions(const _Ptr_base<_Ty2>& _Other) noexcept {
        // implement weak_ptr's copy converting ctor
        if (_Other._Rep) {
            _Rep = _Other._Rep; // always share ownership
            _Rep->_Incwref();

            if (_Rep->_Incref_nz()) {
                _Ptr = _Other._Ptr; // keep resource alive during conversion, handling virtual inheritance
                _Rep->_Decref();
            } else {
                _STL_INTERNAL_CHECK(!_Ptr);
            }
        } else {
            _STL_INTERNAL_CHECK(!_Ptr && !_Rep);
        }
    }

    template <class _Ty2>
    void _Weakly_convert_rvalue_avoiding_expired_conversions(_Ptr_base<_Ty2>&& _Other) noexcept {
        // implement weak_ptr's move converting ctor
        _Rep        = _Other._Rep; // always transfer ownership
        _Other._Rep = nullptr;

        if (_Rep && _Rep->_Incref_nz()) {
            _Ptr = _Other._Ptr; // keep resource alive during conversion, handling virtual inheritance
            _Rep->_Decref();
        } else {
            _STL_INTERNAL_CHECK(!_Ptr);
        }

        _Other._Ptr = nullptr;
    }

    void _Incwref() const noexcept {
        if (_Rep) {
            _Rep->_Incwref();
        }
    }

    void _Decwref() noexcept { // decrement weak reference count
        if (_Rep) {
            _Rep->_Decwref();
        }
    }

private:
    element_type* _Ptr{nullptr};
    _Ref_count_base* _Rep{nullptr};

    template <class _Ty0>
    friend class _Ptr_base;

    friend shared_ptr<_Ty>;

    template <class _Ty0>
    friend struct atomic;

    friend _Exception_ptr_access;

#if _HAS_STATIC_RTTI
    template <class _Dx, class _Ty0>
    friend _Dx* get_deleter(const shared_ptr<_Ty0>& _Sx) noexcept;
#endif // _HAS_STATIC_RTTI
};

~~~

`_Ref_count_base`中维护了两个成员变量（引用计数指针），

~~~cpp
_Atomic_counter_t _Uses = 1;	//当前有多少强指针指向内部指针（element_type）
_Atomic_counter_t _Weaks = 1;	//当前引用计数类对象的使用次数（_Ref_count_base）


class __declspec(novtable) _Ref_count_base { // common code for reference counting
private:
#ifdef _M_CEE_PURE
    // permanent workaround to avoid mentioning _purecall in msvcurt.lib, ptrustu.lib, or other support libs
    virtual void _Destroy() noexcept {
        _CSTD abort();
    }

    virtual void _Delete_this() noexcept {
        _CSTD abort();
    }
#else // ^^^ defined(_M_CEE_PURE) / !defined(_M_CEE_PURE) vvv
    virtual void _Destroy() noexcept     = 0; // destroy managed resource
    virtual void _Delete_this() noexcept = 0; // destroy self
#endif // ^^^ !defined(_M_CEE_PURE) ^^^

    _Atomic_counter_t _Uses  = 1;
    _Atomic_counter_t _Weaks = 1;

protected:
    constexpr _Ref_count_base() noexcept = default; // non-atomic initializations

public:
    _Ref_count_base(const _Ref_count_base&)            = delete;
    _Ref_count_base& operator=(const _Ref_count_base&) = delete;

    virtual ~_Ref_count_base() noexcept {} // TRANSITION, should be non-virtual

    bool _Incref_nz() noexcept { // increment use count if not zero, return true if successful
        auto& _Volatile_uses = reinterpret_cast<volatile long&>(_Uses);
#ifdef _M_CEE_PURE
        long _Count = *_Atomic_address_as<const long>(&_Volatile_uses);
#else
        long _Count = __iso_volatile_load32(reinterpret_cast<volatile int*>(&_Volatile_uses));
#endif
        while (_Count != 0) {
            const long _Old_value = _INTRIN_RELAXED(_InterlockedCompareExchange)(&_Volatile_uses, _Count + 1, _Count);
            if (_Old_value == _Count) {
                return true;
            }

            _Count = _Old_value;
        }

        return false;
    }

    void _Incref() noexcept { // increment use count
        _MT_INCR(_Uses);
    }

    void _Incwref() noexcept { // increment weak reference count
        _MT_INCR(_Weaks);
    }

    void _Decref() noexcept { // decrement use count
        if (_MT_DECR(_Uses) == 0) {
            _Destroy();
            _Decwref();
        }
    }

    void _Decwref() noexcept { // decrement weak reference count
        if (_MT_DECR(_Weaks) == 0) {
            _Delete_this();
        }
    }

    long _Use_count() const noexcept {
        return static_cast<long>(_Uses);
    }

    virtual void* _Get_deleter(const type_info&) const noexcept {
        return nullptr;
    }
};
~~~

因此释放智能指针对象时，需要释放`element_type`和`_Ref_count_base`指针对象。`element_type`的释放是根据`_Ref_count_base`里面`_Uses`来实现的（当值为0释放）。`_Ref_count_base`的释放是根据`_Weaks`来实现的（当值为0释放）。

那么`_Ref_count_base`是如何创建的呢，我们来看`shared_ptr`的源码：

~~~cpp
_EXPORT_STD template <class _Ty>
class shared_ptr : public _Ptr_base<_Ty> { // class for reference counted resource management
private:
    using _Mybase = _Ptr_base<_Ty>;

public:
    using typename _Mybase::element_type;

#if _HAS_CXX17
    using weak_type = weak_ptr<_Ty>;
#endif // _HAS_CXX17

    constexpr shared_ptr() noexcept = default;

    constexpr shared_ptr(nullptr_t) noexcept {} // construct empty shared_ptr

    template <class _Ux,
        enable_if_t<conjunction_v<conditional_t<is_array_v<_Ty>, _Can_array_delete<_Ux>, _Can_scalar_delete<_Ux>>,
                        _SP_convertible<_Ux, _Ty>>,
            int> = 0>
    explicit shared_ptr(_Ux* _Px) { // construct shared_ptr object that owns _Px
        if constexpr (is_array_v<_Ty>) {
            _Setpd(_Px, default_delete<_Ux[]>{});
        } else {
            _Temporary_owner<_Ux> _Owner(_Px);
            _Set_ptr_rep_and_enable_shared(_Owner._Ptr, new _Ref_count<_Ux>(_Owner._Ptr));
            _Owner._Ptr = nullptr;
        }
    }

    template <class _Ux, class _Dx,
        enable_if_t<conjunction_v<is_move_constructible<_Dx>, _Can_call_function_object<_Dx&, _Ux*&>,
                        _SP_convertible<_Ux, _Ty>>,
            int> = 0>
    shared_ptr(_Ux* _Px, _Dx _Dt) { // construct with _Px, deleter
        _Setpd(_Px, _STD move(_Dt));
    }

    template <class _Ux, class _Dx, class _Alloc,
        enable_if_t<conjunction_v<is_move_constructible<_Dx>, _Can_call_function_object<_Dx&, _Ux*&>,
                        _SP_convertible<_Ux, _Ty>>,
            int> = 0>
    shared_ptr(_Ux* _Px, _Dx _Dt, _Alloc _Ax) { // construct with _Px, deleter, allocator
        _Setpda(_Px, _STD move(_Dt), _Ax);
    }

    template <class _Dx,
        enable_if_t<conjunction_v<is_move_constructible<_Dx>, _Can_call_function_object<_Dx&, nullptr_t&>>, int> = 0>
    shared_ptr(nullptr_t, _Dx _Dt) { // construct with nullptr, deleter
        _Setpd(nullptr, _STD move(_Dt));
    }

    template <class _Dx, class _Alloc,
        enable_if_t<conjunction_v<is_move_constructible<_Dx>, _Can_call_function_object<_Dx&, nullptr_t&>>, int> = 0>
    shared_ptr(nullptr_t, _Dx _Dt, _Alloc _Ax) { // construct with nullptr, deleter, allocator
        _Setpda(nullptr, _STD move(_Dt), _Ax);
    }

    template <class _Ty2>
    shared_ptr(const shared_ptr<_Ty2>& _Right, element_type* _Px) noexcept {
        // construct shared_ptr object that aliases _Right
        this->_Alias_construct_from(_Right, _Px);
    }

    template <class _Ty2>
    shared_ptr(shared_ptr<_Ty2>&& _Right, element_type* _Px) noexcept {
        // move construct shared_ptr object that aliases _Right
        this->_Alias_move_construct_from(_STD move(_Right), _Px);
    }

    shared_ptr(const shared_ptr& _Other) noexcept { // construct shared_ptr object that owns same resource as _Other
        this->_Copy_construct_from(_Other);
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    shared_ptr(const shared_ptr<_Ty2>& _Other) noexcept {
        // construct shared_ptr object that owns same resource as _Other
        this->_Copy_construct_from(_Other);
    }

    shared_ptr(shared_ptr&& _Right) noexcept { // construct shared_ptr object that takes resource from _Right
        this->_Move_construct_from(_STD move(_Right));
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    shared_ptr(shared_ptr<_Ty2>&& _Right) noexcept { // construct shared_ptr object that takes resource from _Right
        this->_Move_construct_from(_STD move(_Right));
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    explicit shared_ptr(const weak_ptr<_Ty2>& _Other) { // construct shared_ptr object that owns resource *_Other
        if (!this->_Construct_from_weak(_Other)) {
            _Throw_bad_weak_ptr();
        }
    }

#if _HAS_AUTO_PTR_ETC
    template <class _Ty2, enable_if_t<is_convertible_v<_Ty2*, _Ty*>, int> = 0>
    shared_ptr(auto_ptr<_Ty2>&& _Other) { // construct shared_ptr object that owns *_Other.get()
        _Ty2* _Px = _Other.get();
        _Set_ptr_rep_and_enable_shared(_Px, new _Ref_count<_Ty2>(_Px));
        _Other.release();
    }
#endif // _HAS_AUTO_PTR_ETC

    template <class _Ux, class _Dx,
        enable_if_t<conjunction_v<_SP_pointer_compatible<_Ux, _Ty>,
                        is_convertible<typename unique_ptr<_Ux, _Dx>::pointer, element_type*>>,
            int> = 0>
    shared_ptr(unique_ptr<_Ux, _Dx>&& _Other) {
        using _Fancy_t   = typename unique_ptr<_Ux, _Dx>::pointer;
        using _Raw_t     = typename unique_ptr<_Ux, _Dx>::element_type*;
        using _Deleter_t = conditional_t<is_reference_v<_Dx>, decltype(_STD ref(_Other.get_deleter())), _Dx>;

        const _Fancy_t _Fancy = _Other.get();

        if (_Fancy) {
            const _Raw_t _Raw = _Fancy;
            const auto _Rx =
                new _Ref_count_resource<_Fancy_t, _Deleter_t>(_Fancy, _STD forward<_Dx>(_Other.get_deleter()));
            _Set_ptr_rep_and_enable_shared(_Raw, _Rx);
            _Other.release();
        }
    }

    ~shared_ptr() noexcept { // release resource
        this->_Decref();
    }

    shared_ptr& operator=(const shared_ptr& _Right) noexcept {
        shared_ptr(_Right).swap(*this);
        return *this;
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    shared_ptr& operator=(const shared_ptr<_Ty2>& _Right) noexcept {
        shared_ptr(_Right).swap(*this);
        return *this;
    }

    shared_ptr& operator=(shared_ptr&& _Right) noexcept { // take resource from _Right
        shared_ptr(_STD move(_Right)).swap(*this);
        return *this;
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    shared_ptr& operator=(shared_ptr<_Ty2>&& _Right) noexcept { // take resource from _Right
        shared_ptr(_STD move(_Right)).swap(*this);
        return *this;
    }

#if _HAS_AUTO_PTR_ETC
    template <class _Ty2, enable_if_t<is_convertible_v<_Ty2*, _Ty*>, int> = 0>
    shared_ptr& operator=(auto_ptr<_Ty2>&& _Right) {
        shared_ptr(_STD move(_Right)).swap(*this);
        return *this;
    }
#endif // _HAS_AUTO_PTR_ETC

    template <class _Ux, class _Dx,
        enable_if_t<conjunction_v<_SP_pointer_compatible<_Ux, _Ty>,
                        is_convertible<typename unique_ptr<_Ux, _Dx>::pointer, element_type*>>,
            int> = 0>
    shared_ptr& operator=(unique_ptr<_Ux, _Dx>&& _Right) { // move from unique_ptr
        shared_ptr(_STD move(_Right)).swap(*this);
        return *this;
    }

    void swap(shared_ptr& _Other) noexcept {
        this->_Swap(_Other);
    }

    void reset() noexcept { // release resource and convert to empty shared_ptr object
        shared_ptr().swap(*this);
    }

    template <class _Ux,
        enable_if_t<conjunction_v<conditional_t<is_array_v<_Ty>, _Can_array_delete<_Ux>, _Can_scalar_delete<_Ux>>,
                        _SP_convertible<_Ux, _Ty>>,
            int> = 0>
    void reset(_Ux* _Px) { // release, take ownership of _Px
        shared_ptr(_Px).swap(*this);
    }

    template <class _Ux, class _Dx,
        enable_if_t<conjunction_v<is_move_constructible<_Dx>, _Can_call_function_object<_Dx&, _Ux*&>,
                        _SP_convertible<_Ux, _Ty>>,
            int> = 0>
    void reset(_Ux* _Px, _Dx _Dt) { // release, take ownership of _Px, with deleter _Dt
        shared_ptr(_Px, _Dt).swap(*this);
    }

    template <class _Ux, class _Dx, class _Alloc,
        enable_if_t<conjunction_v<is_move_constructible<_Dx>, _Can_call_function_object<_Dx&, _Ux*&>,
                        _SP_convertible<_Ux, _Ty>>,
            int> = 0>
    void reset(_Ux* _Px, _Dx _Dt, _Alloc _Ax) { // release, take ownership of _Px, with deleter _Dt, allocator _Ax
        shared_ptr(_Px, _Dt, _Ax).swap(*this);
    }

    using _Mybase::get;

    template <class _Ty2 = _Ty, enable_if_t<!disjunction_v<is_array<_Ty2>, is_void<_Ty2>>, int> = 0>
    _NODISCARD _Ty2& operator*() const noexcept {
        return *get();
    }

    template <class _Ty2 = _Ty, enable_if_t<!is_array_v<_Ty2>, int> = 0>
    _NODISCARD _Ty2* operator->() const noexcept {
        return get();
    }

    template <class _Ty2 = _Ty, class _Elem = element_type, enable_if_t<is_array_v<_Ty2>, int> = 0>
    _NODISCARD _Elem& operator[](ptrdiff_t _Idx) const noexcept /* strengthened */ {
        return get()[_Idx];
    }

#if _HAS_DEPRECATED_SHARED_PTR_UNIQUE
    _CXX17_DEPRECATE_SHARED_PTR_UNIQUE _NODISCARD bool unique() const noexcept {
        // return true if no other shared_ptr object owns this resource
        return this->use_count() == 1;
    }
#endif // _HAS_DEPRECATED_SHARED_PTR_UNIQUE

    explicit operator bool() const noexcept {
        return get() != nullptr;
    }

private:
    template <class _UxptrOrNullptr, class _Dx>
    void _Setpd(const _UxptrOrNullptr _Px, _Dx _Dt) { // take ownership of _Px, deleter _Dt
        _Temporary_owner_del<_UxptrOrNullptr, _Dx> _Owner(_Px, _Dt);
        _Set_ptr_rep_and_enable_shared(
            _Owner._Ptr, new _Ref_count_resource<_UxptrOrNullptr, _Dx>(_Owner._Ptr, _STD move(_Dt)));
        _Owner._Call_deleter = false;
    }

    template <class _UxptrOrNullptr, class _Dx, class _Alloc>
    void _Setpda(const _UxptrOrNullptr _Px, _Dx _Dt, _Alloc _Ax) { // take ownership of _Px, deleter _Dt, allocator _Ax
        using _Alref_alloc = _Rebind_alloc_t<_Alloc, _Ref_count_resource_alloc<_UxptrOrNullptr, _Dx, _Alloc>>;

        _Temporary_owner_del<_UxptrOrNullptr, _Dx> _Owner(_Px, _Dt);
        _Alref_alloc _Alref(_Ax);
        _Alloc_construct_ptr<_Alref_alloc> _Constructor(_Alref);
        _Constructor._Allocate();
        _STD _Construct_in_place(*_Constructor._Ptr, _Owner._Ptr, _STD move(_Dt), _Ax);
        _Set_ptr_rep_and_enable_shared(_Owner._Ptr, _STD _Unfancy(_Constructor._Ptr));
        _Constructor._Ptr    = nullptr;
        _Owner._Call_deleter = false;
    }

#if _HAS_CXX20
    template <_Not_builtin_array _Ty0, class... _Types>
    friend shared_ptr<_Ty0> make_shared(_Types&&... _Args);

    template <_Not_builtin_array _Ty0, class _Alloc, class... _Types>
    friend shared_ptr<_Ty0> allocate_shared(const _Alloc& _Al_arg, _Types&&... _Args);

    template <_Bounded_builtin_array _Ty0>
    friend shared_ptr<_Ty0> make_shared();

    template <_Bounded_builtin_array _Ty0, class _Alloc>
    friend shared_ptr<_Ty0> allocate_shared(const _Alloc& _Al_arg);

    template <_Bounded_builtin_array _Ty0>
    friend shared_ptr<_Ty0> make_shared(const remove_extent_t<_Ty0>& _Val);

    template <_Bounded_builtin_array _Ty0, class _Alloc>
    friend shared_ptr<_Ty0> allocate_shared(const _Alloc& _Al_arg, const remove_extent_t<_Ty0>& _Val);

    template <_Not_unbounded_builtin_array _Ty0>
    friend shared_ptr<_Ty0> make_shared_for_overwrite();

    template <_Not_unbounded_builtin_array _Ty0, class _Alloc>
    friend shared_ptr<_Ty0> allocate_shared_for_overwrite(const _Alloc& _Al_arg);

    template <class _Ty0, class... _ArgTypes>
    friend shared_ptr<_Ty0> _Make_shared_unbounded_array(size_t _Count, const _ArgTypes&... _Args);

    template <bool _IsForOverwrite, class _Ty0, class _Alloc, class... _ArgTypes>
    friend shared_ptr<_Ty0> _Allocate_shared_unbounded_array(
        const _Alloc& _Al, size_t _Count, const _ArgTypes&... _Args);
#else // ^^^ _HAS_CXX20 / !_HAS_CXX20 vvv
    template <class _Ty0, class... _Types>
    friend shared_ptr<_Ty0> make_shared(_Types&&... _Args);

    template <class _Ty0, class _Alloc, class... _Types>
    friend shared_ptr<_Ty0> allocate_shared(const _Alloc& _Al_arg, _Types&&... _Args);
#endif // ^^^ !_HAS_CXX20 ^^^

    template <class _Ux>
    void _Set_ptr_rep_and_enable_shared(_Ux* const _Px, _Ref_count_base* const _Rx) noexcept { // take ownership of _Px
        this->_Ptr = _Px;
        this->_Rep = _Rx;
        if constexpr (conjunction_v<negation<is_array<_Ty>>, negation<is_volatile<_Ux>>, _Can_enable_shared<_Ux>>) {
            if (_Px && _Px->_Wptr.expired()) {
                _Px->_Wptr = shared_ptr<remove_cv_t<_Ux>>(*this, const_cast<remove_cv_t<_Ux>*>(_Px));
            }
        }
    }

    void _Set_ptr_rep_and_enable_shared(nullptr_t, _Ref_count_base* const _Rx) noexcept { // take ownership of nullptr
        this->_Ptr = nullptr;
        this->_Rep = _Rx;
    }
};

~~~



一个有效的`weak_ptr`对象一般是通过`shared_ptr`构造的或者是通过拷贝（移动）其他`weak_ptr`对象得到的，`weak_ptr`对象的构造不涉及控制块的创建。

观察`weak_ptr`源码就可以发现，它和`shared_ptr`用的同一套引用计数模块，在堆上维护两个原子变量，只在它构造，拷贝，析构这一系列函数里不对`_Uses`进行修改。

~~~cpp
_EXPORT_STD template <class _Ty>
class weak_ptr : public _Ptr_base<_Ty> { // class for pointer to reference counted resource
public:
#ifndef _M_CEE_PURE
    // When a constructor is converting from weak_ptr<_Ty2> to weak_ptr<_Ty>, the below type trait intentionally asks
    // whether it would be possible to static_cast from _Ty* to const _Ty2*; see N4950 [expr.static.cast]/12.

    // Primary template, the value is used when the substitution fails.
    template <class _Ty2, class = const _Ty2*>
    static constexpr bool _Must_avoid_expired_conversions_from = true;

    // Template specialization, the value is used when the substitution succeeds.
    template <class _Ty2>
    static constexpr bool
        _Must_avoid_expired_conversions_from<_Ty2, decltype(static_cast<const _Ty2*>(static_cast<_Ty*>(nullptr)))> =
            false;
#endif // ^^^ !defined(_M_CEE_PURE) ^^^

    constexpr weak_ptr() noexcept {}

    weak_ptr(const weak_ptr& _Other) noexcept {
        this->_Weakly_construct_from(_Other); // same type, no conversion
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    weak_ptr(const shared_ptr<_Ty2>& _Other) noexcept {
        this->_Weakly_construct_from(_Other); // shared_ptr keeps resource alive during conversion
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    weak_ptr(const weak_ptr<_Ty2>& _Other) noexcept {
#ifdef _M_CEE_PURE
        constexpr bool _Avoid_expired_conversions = true; // slow, but always safe; avoids error LNK1179
#else
        constexpr bool _Avoid_expired_conversions = _Must_avoid_expired_conversions_from<_Ty2>;
#endif

        if constexpr (_Avoid_expired_conversions) {
            this->_Weakly_convert_lvalue_avoiding_expired_conversions(_Other);
        } else {
            this->_Weakly_construct_from(_Other);
        }
    }

    weak_ptr(weak_ptr&& _Other) noexcept {
        this->_Move_construct_from(_STD move(_Other));
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    weak_ptr(weak_ptr<_Ty2>&& _Other) noexcept {
#ifdef _M_CEE_PURE
        constexpr bool _Avoid_expired_conversions = true; // slow, but always safe; avoids error LNK1179
#else
        constexpr bool _Avoid_expired_conversions = _Must_avoid_expired_conversions_from<_Ty2>;
#endif

        if constexpr (_Avoid_expired_conversions) {
            this->_Weakly_convert_rvalue_avoiding_expired_conversions(_STD move(_Other));
        } else {
            this->_Move_construct_from(_STD move(_Other));
        }
    }

    ~weak_ptr() noexcept {
        this->_Decwref();
    }

    weak_ptr& operator=(const weak_ptr& _Right) noexcept {
        weak_ptr(_Right).swap(*this);
        return *this;
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    weak_ptr& operator=(const weak_ptr<_Ty2>& _Right) noexcept {
        weak_ptr(_Right).swap(*this);
        return *this;
    }

    weak_ptr& operator=(weak_ptr&& _Right) noexcept {
        weak_ptr(_STD move(_Right)).swap(*this);
        return *this;
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    weak_ptr& operator=(weak_ptr<_Ty2>&& _Right) noexcept {
        weak_ptr(_STD move(_Right)).swap(*this);
        return *this;
    }

    template <class _Ty2, enable_if_t<_SP_pointer_compatible<_Ty2, _Ty>::value, int> = 0>
    weak_ptr& operator=(const shared_ptr<_Ty2>& _Right) noexcept {
        weak_ptr(_Right).swap(*this);
        return *this;
    }

    void reset() noexcept { // release resource, convert to null weak_ptr object
        weak_ptr{}.swap(*this);
    }

    void swap(weak_ptr& _Other) noexcept {
        this->_Swap(_Other);
    }

    _NODISCARD bool expired() const noexcept {
        return this->use_count() == 0;
    }

    _NODISCARD shared_ptr<_Ty> lock() const noexcept { // convert to shared_ptr
        shared_ptr<_Ty> _Ret;
        (void) _Ret._Construct_from_weak(*this);
        return _Ret;
    }
};
~~~



因为`weak_ptr`的用法就是 `lock`创建一个临时 `shared_ptr`检查`weak_ptr`指向对象的生命周期，需要知道目前真实的引用计数。





弱引用计数的增加可以分为下面几种情况:

- 通过`std::shared_ptr`构造`std::weak_ptr`
- 通过`std::weak_ptr`构造`std::weak_ptr`
- 通过`std::shared_ptr`给`std::weak_ptr`赋值
- 通过`std::weak_ptr`给`std::weak_ptr`赋值

其实本质是靠调用`_M_weak_add_ref()`增加的弱引用计数，

弱引用计数的减少可以分为下面几种情况:

- `std::weak_ptr`析构
- `std::weak_ptr`对象被覆盖（赋值操作覆盖原`std::weak_ptr`）



引用计数的增加可以分为下面几种情况:

- 通过`std::shared_ptr`构造`std::shared_ptr`
- 通过`std::shared_ptr`给`std::shared_ptr`赋值
- `std::weak_ptr`升级为`std::shared_ptr`



[C++智能指针学习——小谈引用计数 - paw5zx - 博客园](https://www.cnblogs.com/paw5zx/p/18120139)

[智能指针std::weak_ptr - 知乎](https://zhuanlan.zhihu.com/p/678652955)

[C++2.0 shared_ptr和weak_ptr深入刨析_c++ weak count-CSDN博客](https://blog.csdn.net/qq_41540355/article/details/123123404)