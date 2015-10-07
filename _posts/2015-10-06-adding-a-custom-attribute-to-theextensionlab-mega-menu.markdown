---
layout: post
title:  "Add custom category attributes to TheExtensionLab_MegaMenu menu"
date:   2015-10-06 15:05:00
categories: jekyll update
tags: featured
---
So you've installed TheExtensionLab_MegaMenu and it's working great. But the guys over in design have put a small icon next to each top level menu item and your boss says he wants those and they need to be configurable. But your not quite sure how to add those in without loading the categories again.

In this post we will show you how pass additional category attributes into the MegaMenu without creating extra load.

The first step is to initialize an empty module that depends on TheExtensionLab_MegaMenu so to do that we create two files below:

app/etc/modules/TheExtensionLab_MenuAttributes.xml

```
<?xml version="1.0"?>
<config>
    <modules>
        <TheExtensionLab_MenuAttributes>
            <active>true</active>
            <codePool>local</codePool>
            <depends>
                <TheExtensionLab_MegaMenu/>
            </depends>
        </TheExtensionLab_MenuAttributes>
    </modules>
</config>
```

app/code/local/TheExtensionLab/MenuAttributes/etc/config.xml

```
<?xml version="1.0"?>
<config>
    <modules>
        <TheExtensionLab_MenuAttributes>
            <version>0.1.0</version>
        </TheExtensionLab_MenuAttributes>
    </modules>
</config>
````

Lets assume that the Categories Image and Thumbnail images are allready utilised by another part of the website and we can't just used those. So first we need to create a new attribute 'icon_image'

We need to first add a setup node to our config.xml file

```
<global>
    <resources>
        <theextensionlab_menuattributes_setup>
            <setup>
                <module>TheExtensionLab_MenuAttribute</module>
                <class>Mage_Catalog_Model_Resource_Setup</class>
            </setup>
        </theextensionlab_menuattributes_setup>
    </resources>
</global>
```

app/code/local/TheExtensionLab/MenuAttributes/sql/theextensionlab\_menuattributes\_setup/install-0.1.0.php

```
<?php
$this->startSetup();
$this->addAttribute(Mage_Catalog_Model_Category::ENTITY, 'icon_image', array(
    'type'                       => 'varchar',
    'label'                      => 'Image',
    'input'                      => 'image',
    'backend'                    => 'catalog/category_attribute_backend_image',
    'required'                   => false,
    'sort_order'                 => 4,
    'global'                     => Mage_Catalog_Model_Resource_Eav_Attribute::SCOPE_STORE,
    'group'                      => 'General',

));
$this->endSetup();
```

If you now refresh your admin (with cache disabled or cleared) you should see a new attribute under the General tab of each category as below
![New Icon Attribute](../../../../../assets/images/menuattributes/new-icon-image.jpg "New Icon Attribute")

In my case i'd like to add small icons next to each image such as the home icon below, so i've uploaded that to the category and now we need to add it to the frontend menu data an render it in our template
![Home Icon](../../../../../assets/images/menuattributes/home-icon-uploaded.jpg "Home Icon")

New category attributes can be added to the MegaMenu using a config xml node "theextensionlab_megamenu/extra_attributes" so in our config.xml we want to add the code below to the config node

```
<theextensionlab_megamenu>
    <extra_attributes>
        <icon_image/>
    </extra_attributes>
</theextensionlab_megamenu>
```

These "extra_attributes" are passed into the Menu select call when the categories are pulled from the database.

Although pulled from the database they are not yet added to our Menu Object, to do that we need to hook into a new event provided by the MegaMenu extension 'menu\_get\_category\_menu\_data\_after'

In our config.xml file we need to add the below to the global node, which defines our models and says on menu\_get\_category\_menu\_data\_after call theextensionlab_menuattributes/observer::MenuGetCategoryMenuDataAfter

```
<models>
    <theextensionlab_menuattributes>
        <class>TheExtensionLab_MenuAttributes_Model</class>
    </theextensionlab_menuattributes>
</models>

<events>
    <menu_get_category_menu_data_after>
        <observers>
            <theextensionlab_menuattributes>
                <class>theextensionlab_menuattributes/observer</class>
                <method>menuGetCategoryMenuDataAfter</method>
            </theextensionlab_menuattributes>
        </observers>
    </menu_get_category_menu_data_after>
</events>
```

We then need to create the observer class
app/code/local/TheExtensionLab/MenuAttributes/Model/Observer.php that will prefix the raw image data from the DB with the media url and pass it to our menu object

```
<?php
class TheExtensionLab_MenuAttributes_Model_Observer
{
    public function menuGetCategoryMenuDataAfter(Varien_Event_Observer $observer)
    {
        $category = $observer->getCategory();
        $categoryMenuData = $observer->getCategoryMenuData();

        if ($category->hasIconImage()){
            $imageUrl = $this->_prefixImageWithMediaPath($category->getIconImage());
            $categoryMenuData->setIconImage($imageUrl);
        }
    }

    private function _prefixImageWithMediaPath($image)
    {
        $mediaPath = Mage::getBaseUrl(Mage_Core_Model_Store::URL_TYPE_MEDIA);
        $categoryImageFolder = 'catalog/category/';
        $imageUrl = $mediaPath.$categoryImageFolder.$image;
        return $imageUrl;
    }
}
```

The next thing we need to override the menus template file by copying it to your theme so copy
app/design/frontend/base/default/template/theextensionlab/megamenu/page/html/topmenu/renderer.phtml to app/design/frontend/[YOUR_PACKAGE]/[YOUR_THEME]/template/theextensionlab/megamenu/page/html/topmenu/renderer.phtml

and at around line 21 inside the category \<a> tag add the following code:

```
<?php if($child->hasIconImage()):?>
    <span>
        <img src="<?php echo $child->getIconImage();?>" />
    </span>
<?php endif;?>
```

The image should then appear in the menu and with a few css tweaks in your theme's css file:

```
img.icon-image{
  display:inline-block;
  position:relative;
  top:2px;
}

```

If you upload a few icons to your categories and refresh the homepage you should end up with something like below, which is the finished product.

![Icon Menu](../../../../../assets/images/menuattributes/icon-menu.jpg "Icon Menu")

***Quick Summary:***

So in summary we created a new category attribute and passed that attribute to the menu using the 'theextensionlab\_megamenu/extra\_attributes' configuration node along with the 'menu\_get\_category\_menu\_data\_after' event. I hope it was useful.