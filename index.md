---
title: Home
layout: home
nav_order: 1
---

# PWN Counter

My CTF writeups (HTB, THM, CyberDefenders, etc.)

<div class="pwn-counter-wrap">
  <div id="pwn-counter" class="pwn-counter-number">0</div>
  <div class="pwn-counter-label">Boxes Pwned</div>
</div>

<style>
.pwn-counter-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin: 2rem 0;
  padding: 1.5rem;
  border: 1px solid #3a3a3a;
  border-radius: 12px;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
}
.pwn-counter-number {
  font-size: 4rem;
  font-weight: 800;
  font-family: 'SFMono-Regular', Consolas, monospace;
  color: #00ff9c;
  text-shadow: 0 0 12px rgba(0, 255, 156, 0.5);
  line-height: 1;
}
.pwn-counter-label {
  margin-top: 0.5rem;
  font-size: 0.9rem;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  color: #8a8a8a;
}
</style>

<script>
(function () {
  // ↓↓↓ UPDATE THIS NUMBER EVERY TIME YOU SOLVE A NEW CHALLENGE ↓↓↓
  var PWN_COUNT = 0;
  // ↑↑↑ UPDATE THIS NUMBER EVERY TIME YOU SOLVE A NEW CHALLENGE ↑↑↑

  var el = document.getElementById('pwn-counter');
  if (!el) return;

  var duration = 1200; // ms, total animation time
  var start = null;

  function step(timestamp) {
    if (!start) start = timestamp;
    var progress = Math.min((timestamp - start) / duration, 1);
    // ease-out for a satisfying "settle" at the end
    var eased = 1 - Math.pow(1 - progress, 3);
    var current = Math.floor(eased * PWN_COUNT);
    el.textContent = current;
    if (progress < 1) {
      requestAnimationFrame(step);
    } else {
      el.textContent = PWN_COUNT;
    }
  }

  if (PWN_COUNT > 0) {
    requestAnimationFrame(step);
  }
})();
</script>

Use the navigation on the left (or the search bar above) to browse by category:

- **Web Exploitation** — SQLi, XSS, SSRF, auth bypass, etc.
- **Pwn / Binary Exploitation** — buffer overflows, ROP chains, heap exploitation.
- **Reverse Engineering** — disassembly, decompilation, obfuscation.
- **Cryptography** — classical and modern crypto challenges.
- **Forensics** — memory dumps, packet captures, steganography.
- **Misc** — everything else (OSINT, scripting puzzles, etc).

Each writeup includes the challenge description, my approach, the tools used, and the full solution leading to the flag.

---

*New writeups are added as I solve more challenges. Feel free to browse via the sidebar.*
