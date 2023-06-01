---
title: Introducing Rich Media Embeds Addon for Bookstack
date: 2023-05-29 11:58:00 +300
categories: [homelab, showcase, tutorial]
tags: [bookstack, js, addon, custom-solution]
img_path: /posts/2023-05-29-introducing-rich-media-embeds-addon-for-bookstack/
image:
  path: embedded-image-uh2tjsf0.jpeg
  #alt: Article Header
thumb:
  path: embedded-image-uh2tjsf0.jpeg
---

# Rich Media Embeds

> [The code can be found here](https://github.com/tenekev/bookstack-rich-media-embeds)
{: .prompt-info }

## 🤔 Why?

Rich Media Embeds is a mod for Bookstack. It allows users to add more than just text links to their documentations. In modern note-taking, everyone and their mother has some sort of Rich Media embeds, implemented in their product - Videos, galleries, code blocks. Bookstack offers these but I felt it's missing content like links, previews for pdf, docx, pptx, xlsx, Unplash integration and so on. None of them are critical but they bridge gaps and streamline workflows, letting you focus on writing.

This first version of Rich Media Embeds focuses on **links**. You click a button, supply an URL and an image appears on the page. The image has the link's **thumbnail**, the **title** and a **description**. Upon saving the page, this image is uploaded into BookStack. It shows on all export types - HTML, PDF, MD, etc.

## 🎯 The objectives are pretty straightforward:

- Create a simple, self-contained item
- Be persistent on the page (present both in read and edit mode)
- Be exportable
- Stored into BookStack's storage
- Completely decoupled from the backend

## ⚙️ How does RME work?

Upon loading the WYSIWYG editor, a button is hooked into the toolbar - `{;}`. From there you can posts URLs that need to be enriched. When you submit an URL, RME inserts a blank link object into the page and awaits for another function to fetch the URL data. RME relies on two types of data. Most modern sites supply [ OpenGraph](https://ogp.me/) meta tags with `og:title`, `og:description` and `og:image`. If these aren't present, it fetches the `<title>`, `<meta property="description">` and the biggest image off the page.

Since everything is done client-side, fetching resources from other domains is an issue. Browsers enforce [Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) and normally what we want to do - fetch another page &amp; its resources, throws a [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) error. To alleviate this issue, RME relays on a CORS Proxy - [CORS-Anywhere](https://github.com/Rob--W/cors-anywhere). You can add a public one if you find one, but we will go over how to set up a private one in Docker.

Once the resources are fetched, RME can begin generating the image. A hidden DOM element is attached to the page where the card is stylized with CSS. Then the HTML DOM element is turned into an SVG, which is turned into a `<canvas>`, which is turned into a base64 string. This metamorphosis is handled by a library called [html2canvas](https://github.com/niklasvh/html2canvas). It sounds convoluted but it's actually the best way to leverage browser rendering into image creation. The base64 string is embedded into the blank link object that was created in the beginning of this madness and you see a neat image like this:

[![https://realpython.com/tutorials/docker/](embedded-image-uh2tjsf0.jpeg)](https://realpython.com/tutorials/docker/)

But why do we need a base64 string? Well because when you save a page in BookStack, it checks the content for embedded base64 strings and uploads them as images into its storage automatically. No need to upload images manually, no API calls, no backend edits. We leverage Bookstack's functionality, thus generating a card for our link. Neat, huh?

## ⛓️ Dependencies

- Bookstack (obviously)
- [TinyMCE](https://www.tiny.cloud/) - The Bookstack WYSIWYG editor
- [CORS-Anywhere Docker image by testcab](https://github.com/testcab/docker-cors-anywhere)
- [html2canvas project by niklasvh](https://github.com/niklasvh/html2canvas)

## 📦 How to install Rich Media Embeds addon

RME is designed to run entirely client-side but this comes with its limitations. The setup process is two-part.

### 1. Setting up[ CORS-Anywhere](https://github.com/Rob--W/cors-anywhere) service:

CORS-A is a lightweight service that even has public instances. It's advised to set up your own without the limitations or risks of public ones. My Bookstack is containerized, thus I'm showing a containerized version of CORS-A too. We will be using [this well-known image](https://hub.docker.com/r/testcab/cors-anywhere) of CORS-A in a snipped of a docker-compose file.

```yaml
version: '3.4'

services:
  cors-anywhere:
    image: testcab/cors-anywhere
    container_name: cors-anywhere
    ports:
      - 8080:8080
    restart: unless-stopped
```

It's tempting to put both containers on the same docker network and try to use `<a href="http://cors-anywhere:8080">http://cors-anywhere:8080</a>` but it will not work. We are running client-side! While the services will see each other, the clients (our browsers) are not part of this network. At the very least, you need to use the docker host ip or hostname like so - `<a href="http://cors-anywhere:8080">http://dockerhost:8080</a>`

> If your Bookstack instance is running on HTTPS, so must CORS-A. Your browser won't allow you to serve insecure HTTP requests on a secured HTTPS page!
{: .prompt-warning }

### 2. Injecting RME addon into Bookstack

Bookstack conveniently allows for the injection of custom code in the `<head>` element of the site. **Settings &gt; Customization &gt; Custom HTML Head Content**.

**The simplest approach** is to just copy the contents of addon-rich-media-embeds.js, html2canvas.min.js and styles.css in respective `<script>` and `<style>` tags. 
[Here is the code](https://github.com/tenekev/bookstack-rich-media-embeds/blob/main/head-simple.html).
However, it introduces a lot of lines of code that is hard to manage.

**The better approach** is to link these files from your filesystem. Here is how to place them in Bookstack's `/config/www/uploads/` directory. This will serve them with the rest of Bookstack's files.

```yaml
version: '3.4'

services: 
  bookstack:
    image: lscr.io/linuxserver/bookstack:latest
    ...
    volumes:
      - ${DIR_CONFIG}/bookstack/config:/config
      # Custom files
      - ${DIR_CONFIG}/bookstack/html2canvas.min.js:/config/www/uploads/html2canvas.min.js
      - ${DIR_CONFIG}/bookstack/addon-rich-media-embeds.js:/config/www/uploads/addon-rich-media-embeds.js
      - ${DIR_CONFIG}/bookstack/styles.css:/config/www/uploads/styles.css
```

Here is how the **Custom HTML Head Content** will look:

```html
<!-- RME Config -->
  <script>
    const proxy_server = 'https://cors.yourdomain.tld/'  // <- Note the trailing slash /
    const remove_tempCard = true;
  </script>

<!-- Link html2canvas from CDN or from file -->
  <script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<!-- OR  -->
  <script src="https://bookstack.yourdomain.tld/uploads/html2canvas.min.js"></script>

<!-- Link RME's JS and CSS -->
  <script src="https://bookstack.yourdomain.tld/uploads/addon-rich-media-embeds.js"></script>
  <link href="https://bookstack.yourdomain.tld/uploads/styles.css" rel="stylesheet"/>
```

> Regardless of your methodology, make sure to change the `proxy_server` variable on line #3 to your CORS-Anywhere instance's address.
{: .prompt-tip }

Once you open the editor, you should see this in your toolbar:

![image.png](image.png) 

You are ready to start posting links for beautiful embeds.

## Examples of RME cards from different sites

Notes in **BOLD** with *Lorem ipsum* as filler.

*Et et aliquip delenit aliquam nulla. Iriure nulla amet ut. Gubergren accusam no molestie dolore sed. Kasd suscipit ea eirmod consetetur kasd nisl aliquyam magna sadipscing et dolor et sadipscing sea. Stet dolor lorem accusam clita erat sit sed commodo amet labore est ut velit rebum dolore facilisis. Sed no luptatum consequat et eos et nihil dolor et consetetur dolore sit labore ipsum.*

#### A blog  


[![https://realpython.com/tutorials/docker/](embedded-image-8xssouyv.jpeg)](https://realpython.com/tutorials/docker/)

*Diam vel takimata exerci lorem dolore sit at exerci ipsum lorem delenit consetetur dolor sed amet. Ipsum ipsum amet. Gubergren sea in est eu dolore at illum dolores aliquyam. Ipsum iriure est dolore at dolores ut molestie eum takimata sit vero labore. Ea no elitr sed sanctus vel aliquyam consequat tation est et esse nisl molestie. Dolore nam elitr molestie elit gubergren amet nonumy.*

#### Another Blog - freecodecamp.org

*Gubergren lorem augue et. Ipsum elitr sea exerci stet voluptua praesent amet te no at wisi. Gubergren voluptua nonumy at at. Ut diam placerat amet facilisis et liber veniam dolore magna dolor sanctus nonumy dolor dolore. Facilisis iusto clita nonummy ipsum imperdiet duo. Kasd dolore lorem diam. Sit dolor dolor lorem esse aliquip ut elit aliquyam ex elitr takimata consectetuer labore.*

[![https://www.freecodecamp.org/news/aws-certified-cloud-practitioner-certification-study-course-pass-the-exam/](embedded-image-pynp6msl.jpeg)](https://www.freecodecamp.org/news/aws-certified-cloud-practitioner-certification-study-course-pass-the-exam/)

#### Medium.com

*Stet rebum est ipsum aliquyam nisl duis sanctus ipsum ipsum duo sit et justo voluptua amet aliquyam consetetur. Vero consectetuer stet ut justo et dolore invidunt aliquyam. Ipsum ipsum aliquyam aliquip dolore cum. Diam ea nonumy accusam ut amet sed sit duo nulla ut vel sit tempor voluptua stet sit eum. Sit sed et et sea aliquyam ipsum sadipscing amet lorem eirmod diam ut. Sanctus consetetur et lorem sea nonumy vulputate aliquip ut nonumy takimata nulla. Nonumy consetetur sit eos labore eos.*

#### A website without OG: data

[![https://medium.com/geekculture/the-most-simplified-integration-of-ansible-and-terraform-49f130b9fc8](embedded-image-oc7p86lo.jpeg)](https://medium.com/geekculture/the-most-simplified-integration-of-ansible-and-terraform-49f130b9fc8)

**In this case RME relies on the biggest image on the page to display as thumbnail.**

*Praesent elitr sea dolore magna sed ea sit molestie dolore clita sit duo kasd erat elitr. Dolor tempor nulla vel duo consetetur consetetur no labore labore eos et enim vel. Diam illum eu duis tempor et voluptua sit eos lorem duo eu diam takimata. Justo sit sit ut facilisis. Ea gubergren duo magna. Vero ut sanctus sed justo. Sed sea stet est sanctus delenit.*

[![https://home.pearsonvue.com/Test-takers.aspx](embedded-image-1cqvvkao.jpeg)](https://home.pearsonvue.com/Test-takers.aspx)

*Diam vel takimata exerci lorem dolore sit at exerci ipsum lorem delenit consetetur dolor sed amet. Ipsum ipsum amet. Gubergren sea in est eu dolore at illum dolores aliquyam. Ipsum iriure est dolore at dolores ut molestie eum takimata sit vero labore. Ea no elitr sed sanctus vel aliquyam consequat tation est et esse nisl molestie. Dolore nam elitr molestie elit gubergren amet nonumy.*

#### An AliExpress Item

[![https://www.aliexpress.com/item/32373860543.html?spm=a2g0o.cart.0.0.71b938daTui1cG&mp=1](embedded-image-9xhwxvjq.jpeg)](https://www.aliexpress.com/item/32373860543.html?spm=a2g0o.cart.0.0.71b938daTui1cG&mp=1)

#### An Amazon Item

**They don't serve OG: data and scraping is harder.**

*Sit sed et et sea aliquyam ipsum sadipscing amet lorem eirmod diam ut. Sanctus consetetur et lorem sea nonumy vulputate aliquip ut nonumy takimata nulla.*

[![https://www.amazon.com/Seagate-FireCuda-Internal-Solid-State/dp/B0977K2C74](embedded-image-8zgu8tca.jpeg)](https://www.amazon.com/Seagate-FireCuda-Internal-Solid-State/dp/B0977K2C74)

#### Some random website

**Again, it lacks OG: data so the biggest image is used instead.**

*Stet rebum est ipsum aliquyam nisl duis sanctus ipsum ipsum duo sit et justo voluptua amet aliquyam consetetur. Vero consectetuer stet ut justo et dolore invidunt aliquyam. Ipsum ipsum aliquyam aliquip dolore cum. Diam ea nonumy accusam ut amet sed sit duo nulla ut vel sit tempor voluptua stet sit eum. Sit sed et et sea aliquyam ipsum sadipscing amet lorem eirmod diam ut. Sanctus consetetur et lorem sea nonumy vulputate aliquip ut nonumy takimata nulla. Nonumy consetetur sit eos labore eos.*

[![https://www.cpubenchmark.net/compare/3454vs3237vs3231vs3260vs2910/Intel-i5-9500T-vs-Intel-i5-8500T-vs-Intel-i5-8400T-vs-Intel-i5-7500-vs-Intel-i7-8750H](embedded-image-gcao6hmu.jpeg)](https://www.cpubenchmark.net/compare/3454vs3237vs3231vs3260vs2910/Intel-i5-9500T-vs-Intel-i5-8500T-vs-Intel-i5-8400T-vs-Intel-i5-7500-vs-Intel-i7-8750H)

*Diam vel takimata exerci lorem dolore sit at exerci ipsum lorem delenit consetetur dolor sed amet. Ipsum ipsum amet. Gubergren sea in est eu dolore at illum dolores aliquyam. Ipsum iriure est dolore at dolores ut molestie eum takimata sit vero labore. Ea no elitr sed sanctus vel aliquyam consequat tation est et esse nisl molestie. Dolore nam elitr molestie elit gubergren amet nonumy.*

**Same site but this page has a bigger image.**

[![https://www.cpubenchmark.net/](embedded-image-mnuusuaq.jpeg)](https://www.cpubenchmark.net/)

#### A Documentation website  

*Praesent elitr sea dolore magna sed ea sit molestie dolore clita sit duo kasd erat elitr. Dolor tempor nulla vel duo consetetur consetetur no labore labore eos et enim vel. Diam illum eu duis tempor et voluptua sit eos lorem duo eu diam takimata. Justo sit sit ut facilisis. Ea gubergren duo magna. Vero ut sanctus sed justo. Sed sea stet est sanctus delenit.*

[![https://developer.hashicorp.com/terraform/tutorials/aws-get-started](embedded-image-nbj20ek3.jpeg)](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)

#### YouTube video

*Accusam magna dolor sed diam tempor et takimata vero eu quod iusto ea dolore sed et dolores duo. Labore dolore dolore ex dolor et iriure no eros diam in justo at diam sit et sit te dolore.*

[![https://www.youtube.com/watch?v=7hn4Ttfq8y0](embedded-image-1pw3qvwq.jpeg)](https://www.youtube.com/watch?v=7hn4Ttfq8y0)

*Sed et invidunt qui elitr sit clita amet ipsum. Zzril eos amet eu feugiat diam nonumy et dolore dolor. No vulputate amet magna consequat ut sed duo sadipscing ipsum euismod. Diam vulputate at kasd augue duo duo clita consectetuer lorem et. Justo molestie justo gubergren vulputate at labore aliquyam sit sit justo eu elitr hendrerit sed. Accusam magna dolor sed diam tempor et takimata vero eu quod iusto ea dolore sed et dolores duo. Labore dolore dolore ex dolor et iriure no eros diam in justo at diam sit et sit te dolore.*

#### An item on Ebay-like market  

<details id="bkmrk--17"><summary>Embed in Spoiler</summary>

[![https://www.olx.bg/d/ad/mini-kompyutar-lenovo-i5-8400t-8gb-250ssd-CID632-ID8U3ME.html?reason=observed_ad](embedded-image-zdrqqgl8.jpeg)](https://www.olx.bg/d/ad/mini-kompyutar-lenovo-i5-8400t-8gb-250ssd-CID632-ID8U3ME.html?reason=observed_ad)

</details>*Sed et invidunt qui elitr sit clita amet ipsum. Zzril eos amet eu feugiat diam nonumy et dolore dolor. No vulputate amet magna consequat ut sed duo sadipscing ipsum euismod. Diam vulputate at kasd augue duo duo clita consectetuer lorem et. Justo molestie justo gubergren vulputate at labore aliquyam sit sit justo eu elitr hendrerit sed. Accusam magna dolor sed diam tempor et takimata vero eu quod iusto ea dolore sed et dolores duo. Labore dolore dolore ex dolor et iriure no eros diam in justo at diam sit et sit te dolore.*

#### GitHub Cards

*Dolore at in diam. Aliquyam imperdiet justo nonummy diam vero vel vulputate vero sed sed ipsum erat. Et lorem eos no takimata sit amet dolor sadipscing at vero ut amet. Amet invidunt stet lorem kasd est eum ea ullamcorper vero gubergren ipsum tincidunt takimata aliquyam takimata illum sit. Duo diam esse gubergren dolores nonummy te et duis blandit takimata at sadipscing vero enim at. Sit no ullamcorper sit. Ad ea ad velit kasd sadipscing accusam molestie no sed autem amet kasd veniam tation et takimata.*

---

#### Repos

[![BookStackApp/BookStack](embedded-image-s0ksiwx8.png)](https://github.com/BookStackApp/BookStack)

*Lorem eum amet labore ea. Erat sanctus magna nonumy nibh sadipscing vulputate takimata placerat elitr consetetur nibh justo elitr eleifend erat. Aliquyam amet erat minim hendrerit nam rebum ipsum elitr soluta ut dolor magna dolore. Consequat quis est kasd magna. Sit et sit no ipsum dolor magna vel ea ea justo lorem justo et vero ipsum suscipit ea. Et in in justo ut gubergren facilisis aliquyam lorem ut option amet dolor erat eirmod et sed sed.*

---

#### Issues

[![BookStackApp/BookStack/issues/705#issuecomment-1203691993](embedded-image-kbuunfir.png)](https://github.com/BookStackApp/BookStack/issues/705#issuecomment-1203691993)

*Accusam eum et tempor nulla kasd. Et consequat et diam no feugiat takimata congue stet et dolore nonumy eirmod tempor. Eirmod et duis labore eos amet at magna. Eu nulla iusto aliquyam suscipit nulla dolor eirmod eu sed zzril sit labore facilisis diam id euismod sea. Duo takimata facilisi rebum lorem veniam eum sit lorem lorem consequat feugiat lorem dolore. Sea sanctus gubergren nonumy takimata erat et accusam dolore eos qui accusam lorem dolore te dolore et sanctus labore.*

---

#### Pull Requests

[![BookStackApp/BookStack/pull/4163](embedded-image-mg6jcn3h.png)](https://github.com/BookStackApp/BookStack/pull/4163)

*Nonumy sadipscing possim sit et dolores vero wisi. Sanctus magna ex nam et exerci voluptua kasd. Delenit laoreet erat eos consequat sea dolore. Ut accumsan ipsum accusam. Consetetur magna erat est erat gubergren consetetur sed. Ad ipsum stet est clita dolore blandit ea dolor nam et amet dolore eu tempor et lobortis magna. Molestie sadipscing luptatum lorem dolore adipiscing nam voluptua nonumy erat dolore nonummy et duis et gubergren tempor diam.*

---

#### Commits

[![BookStackApp/BookStack/pull/4163/commits/de0c572785eb34453625daaf8982172e14f50777](embedded-image-wilnqfe0.png)](https://github.com/BookStackApp/BookStack/pull/4163/commits/de0c572785eb34453625daaf8982172e14f50777)

*Tempor kasd aliquyam est iriure sit justo lorem tempor in. Autem invidunt sit consequat soluta stet eos ipsum zzril dolor sed sadipscing at sit voluptua elitr consetetur possim commodo. Takimata duo vero lorem praesent no illum nisl no sed illum autem justo sed. Magna ipsum et duo luptatum ipsum et sed amet dolores aliquam diam vero. Voluptua eu sea ea hendrerit tempor dolor iriure. Amet et eirmod erat accusam vulputate nisl duis vero duo. Sea nulla et nulla. Ut dolor accusam feugiat clita aliquyam consequat amet at dolor eum amet illum gubergren sea at nisl ea ipsum. Diam ipsum sit.*