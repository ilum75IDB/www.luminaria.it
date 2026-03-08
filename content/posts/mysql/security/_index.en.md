---
title: "Access & Control"
description: "MySQL and MariaDB security in practice: the user@host model, granular privileges and the pitfalls of an authentication system that ties identity to connection origin."
layout: "list"
image: "security.cover.jpg"
---
In MySQL, security begins before the password.<br>
It begins with the question: where are you connecting from?<br>

Because MySQL does not just ask who you are.<br>
It asks who you are **and where you come from**.<br>
And the answer changes everything: privileges, access, even your existence as a user.<br>

It is a model that other databases do not have.<br>
It is powerful when you master it.<br>
It is a trap when you ignore it.<br>

Here you will find analysis of the MySQL and MariaDB authentication model, the most common mistakes in access management and the operational differences between the two engines that anyone working in production needs to know.<br>

Because most MySQL security problems do not come from external attacks.<br>
They come from users created without understanding how the engine interprets them.
