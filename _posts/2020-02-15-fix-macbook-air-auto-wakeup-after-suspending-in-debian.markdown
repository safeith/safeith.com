---
layout: post
title:  "Fix MacBook Air Auto Wakeup After Suspending In Debian"
date:   2020-02-15 13:54:00 +0100
categories: Debian
---
Recently my wife gave her old MacBook Air to me, and I started to test some of the Linux distributions, and finally, I decided to continue with Debian. Debian works for me as well as macOS, and I can say it is more powerful than macOS as the unofficial OS for this hardware. I just had a small issue with Debian on this Macbook Air device, and it was "auto wakeup after suspending" and I fixed it with creating a Systemd unit

I only need to create this Systemd unit
{% highlight bash %}
sudo vim /etc/systemd/system/suspend-fix.service
{% endhighlight %}

Whit this content
{% highlight bash %}
[Unit]
Description=Fix for the suspend issue

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo XHC1 > /proc/acpi/wakeup && echo LID0 > /proc/acpi/wakeup"

[Install]
WantedBy=multi-user.target
{% endhighlight %}

And finally I enabled it
{% highlight bash %}
sudo systemctl enable --now suspend-fix.service
{% endhighlight %}


---
Source: [reddit.com](https://www.reddit.com/r/Fedora/comments/66rwng/here_is_a_solution_to_the_problem_of_macbook_air/)
