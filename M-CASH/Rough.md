/admin/deposit/subscriber/cash-out-store

```html
<ul class="m-nav m-nav--skin-light">

<li class="m-nav__section m--hide">

<span class="m-nav__section-text">Section</span>

</li>

<!-- NEW: My Information -->

<li class="m-nav__item">

<a th:href="@{/admin/my-profile}" class="m-nav__link">

<i class="m-nav__link-icon flaticon-user"></i>

<span class="m-nav__link-text">My Information</span>

</a>

</li>

<!-- NEW: Customer Support -->

<li class="m-nav__item">

<a th:href="@{/admin/customer-support}" class="m-nav__link">

<i class="m-nav__link-icon flaticon-support"></i>

<span class="m-nav__link-text">Customer Support</span>

</a>

</li>

<!-- NEW: Language -->

<li class="m-nav__item">

<a th:href="@{/admin/language}" class="m-nav__link">

<i class="m-nav__link-icon flaticon-globe"></i>

<span class="m-nav__link-text">Language</span>

</a>

</li>

<!-- NEW: About Us -->

<li class="m-nav__item">

<a th:href="@{/admin/about-us}" class="m-nav__link">

<i class="m-nav__link-icon flaticon-information"></i>

<span class="m-nav__link-text">About Us</span>

</a>

</li>

<!-- EXISTING (KEEP AS IS) -->

<li class="m-nav__item">

<a href="/admin/change-password" th:if="${@authService.isAdmin()}" class="m-nav__link">

<i class="m-nav__link-icon flaticon-share"></i>

<span class="m-nav__link-text">Change Password</span>

</a>

</li>

<li class="m-nav__item">

<a href="/admin/subscriber/change-own-pincode" th:if="${@authService.isSubscriber()}" class="m-nav__link">

<i class="m-nav__link-icon flaticon-share"></i>

<span class="m-nav__link-text">Change PinCode</span>

</a>

</li>

<li class="m-nav__separator m-nav__separator--fit"></li>

<!-- Logout -->

<li class="m-nav__item">

<a th:href="@{/admin/logout}" class="btn m-btn--pill btn-secondary m-btn m-btn--custom m-btn--label-brand m-btn--bolder">

Logout

</a>

</li>

</ul>
```