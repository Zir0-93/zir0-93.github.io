---
layout: page
title: Contact
permalink: /contact/
---

<form action="https://formspree.io/f/xvzlobwe" method="POST" class="contact-form" style="max-width: 500px; margin: 0 auto;">
  <p style="margin-bottom: 24px;">Have a question about ML infrastructure, neurosymbolic systems, or want to collaborate? Send me a message.</p>

  <div style="margin-bottom: 16px;">
    <label for="name" style="display: block; margin-bottom: 6px; font-weight: 500;">Name</label>
    <input type="text" id="name" name="name" required
      style="width: 100%; padding: 10px 12px; border: 1px solid #ccc; border-radius: 6px; font-size: 15px; font-family: inherit;">
  </div>

  <div style="margin-bottom: 16px;">
    <label for="email" style="display: block; margin-bottom: 6px; font-weight: 500;">Email</label>
    <input type="email" id="email" name="email" required
      style="width: 100%; padding: 10px 12px; border: 1px solid #ccc; border-radius: 6px; font-size: 15px; font-family: inherit;">
  </div>

  <div style="margin-bottom: 16px;">
    <label for="message" style="display: block; margin-bottom: 6px; font-weight: 500;">Message</label>
    <textarea id="message" name="message" rows="5" required
      style="width: 100%; padding: 10px 12px; border: 1px solid #ccc; border-radius: 6px; font-size: 15px; font-family: inherit; resize: vertical;"></textarea>
  </div>

  <button type="submit"
    style="background: #2563eb; color: white; padding: 10px 20px; border: none; border-radius: 6px; font-size: 15px; font-weight: 500; cursor: pointer; transition: background 0.2s;"
    onmouseover="this.style.background='#1d4ed8'"
    onmouseout="this.style.background='#2563eb'">Send</button>
</form>

<p style="text-align: center; margin-top: 32px; font-size: 0.9em; opacity: 0.7;">
  You can also find me on <a href="https://github.com/{{ site.author.github_username }}" target="_blank" rel="noopener">GitHub</a>
  or <a href="https://www.linkedin.com/in/{{ site.author.linkedin_username }}" target="_blank" rel="noopener">LinkedIn</a>.
</p>
