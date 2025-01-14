---
layout: default 
group: install_trouble
subgroup: 03_install
title: During installation, Reflection Exception error
menu_title: During installation, Reflection Exception error
menu_node: 
menu_order: 25
version: 2.0
github_link: install-gde/trouble/tshoot_wrong-mysql.md
redirect_from: /guides/v1.0/install-gde/trouble/tshoot_wrong-mysql.html
---

<h2 id="install-trouble-clear-var">During installation, Reflection Exception error</h2>

### Details

During the installation, a  message similar to the following displays: 

	[ERROR] exception 'ReflectionException' with message 'Class Magento\Framework\StoreManagerInterface does not exist' 
	in /<path>/lib/internal/Magento/Framework/Code/Reader/ClassReader.php

### Solution

Clear all directories and files under Magento's `var` subdirectory and install the Magento software again.

As the <a href="{{page.baseurl}}install-gde/prereq/file-sys-perms-over.html">Magento file system owner</a> or as a user with `root` privileges, enter the following commands:

	cd <your Magento install directory>/var
	rm -rf cache/* di/* generation/* page_cache/*

