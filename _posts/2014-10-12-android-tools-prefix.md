---
layout: post
title:  "Android tools prefix"
date:   2014-10-12
summary:    When implementing views in Android, you oftentimes want to have values that are only visible in design time.
tags:   android xml designer
---

When implementing views in Android, you oftentimes want to have values that are only meant to render nicely in the layout designer.

This is where the `tools:` prefix comes in handy!

{% highlight xml %}

<TextView
  android:id="@+id/someLabel"
  style="@style/Text.Label"
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"

  tools:text="@string/some_string"
  />

<ListView
  android:id="@+id/someList"
  android:layout_width="match_parent"
  android:layout_height="match_parent"

  tools:listitem="@android:layout/simple_list_item_2"
  />
{% endhighlight %}

In the first text view you can see a text attribute (`tools:text`) that is only visible in the layout designer. This way, you
can provide a sample text that will render nicely in the designer, but not accidentally expose sample texts in your actual application.

In the list view, the `tools:listitem` directive specifies a layout which the list view items should use. This way, your list views are pre-filled with sample items in the designer. If you also use the `tools` directives in your list item layout, the pre-filled items will use them as well.

You can use almost any `android` attribute and replace it with `tools` to have design-time attributes. In fact, if you provide attributes with both prefixes, the `tools` prefix is valid during design-time, while your actual `android` attribute is used during runtime.

You can find more information in the [official documentation](http://tools.android.com/tech-docs/tools-attributes).
