// window.hatchable.config — browser-side helpers for declared values
// and the auto-modal runtime that catches SetupRequired responses.
//
// Auto-injected into every project's <head> by DeployService::headInjection.
// Two responsibilities, in order of importance:
//
// 1. Intercept 412 setup_required responses anywhere in the app.
//    The SDK throws SetupRequired server-side; the gateway returns
//    412 with { hint, key, scope, setup_url }. The deployed app's
//    fetch() calls go through whatever code the agent wrote — which
//    may not handle 412. We monkey-patch window.fetch so any 412 with
//    hint=setup_required gets interpreted: show the modal, paste the
//    value, then transparently retry the original request.
//
// 2. Expose a tiny imperative API:
//      window.hatchable.config.save(key, value, opts?)
//      window.hatchable.config.prompt(key)   // open the modal manually
//
//    Use these when the agent wants their own UX. save() POSTs to
//    /__hatchable/secrets/{key} (the existing platform endpoint) so
//    the secret never touches agent-rendered DOM beyond pass-through.
//
// Styling matches the rest of the platform's neutral cards. Templates
// can override via custom CSS targeting `.hatchable-config-modal`.

(function () {
  if (window.hatchable && window.hatchable.config) return;
  window.hatchable = window.hatchable || {};

  const cfg = (window.__HATCHABLE__ = window.__HATCHABLE__ || {});
  const SLUG = cfg.slug || '';

  // ---------------------------------------------------------------------
  // session-scoped suppression — once the user picks "don't show again
  // this session" for a key, stop popping the modal for it. Keyed PER KEY
  // (one app may surface several different missing keys; silencing one
  // shouldn't hide the others). Backed by sessionStorage so it clears when
  // the tab closes; falls back to an in-memory set when storage throws
  // (private mode, sandboxed iframe).
  // ---------------------------------------------------------------------
  const DISMISS_STORE = 'hatchable:setup-dismissed';
  const memDismissed = new Set();

  function isDismissed(key) {
    if (memDismissed.has(key)) return true;
    try {
      const raw = window.sessionStorage.getItem(DISMISS_STORE);
      return raw ? JSON.parse(raw).includes(key) : false;
    } catch { return false; }
  }
  function markDismissed(key) {
    memDismissed.add(key);
    try {
      const raw = window.sessionStorage.getItem(DISMISS_STORE);
      const arr = raw ? JSON.parse(raw) : [];
      if (!arr.includes(key)) {
        arr.push(key);
        window.sessionStorage.setItem(DISMISS_STORE, JSON.stringify(arr));
      }
    } catch { /* memDismissed already covers this session */ }
  }

  // Append a `from` so the console's "Continue to app →" can round-trip
  // the owner back here after they paste. Mirrors the old auto-redirect.
  function withFrom(url) {
    try {
      const u = new URL(url, window.location.origin);
      if (!u.searchParams.has('from')) u.searchParams.set('from', window.location.href);
      return u.toString();
    } catch { return url; }
  }

  // ---------------------------------------------------------------------
  // modal scheduling — ONE modal on screen at a time, and concurrent 412s
  // for the SAME key share a single dialog (a page firing five parallel
  // reads of one unset key must not stack five identical modals). `chain`
  // serializes distinct keys; `inFlight` coalesces same-key callers.
  // ---------------------------------------------------------------------
  let chain = Promise.resolve();
  const inFlight = new Map();

  function requestModal(descriptor) {
    const key = descriptor.key;
    if (inFlight.has(key)) return inFlight.get(key);
    const p = chain.then(() => openModal(descriptor));
    chain = p.catch(() => {});               // a thrown modal must not wedge the queue
    inFlight.set(key, p);
    const done = () => inFlight.delete(key);
    p.then(done, done);
    return p;
  }

  // ---------------------------------------------------------------------
  // imperative API
  // ---------------------------------------------------------------------

  // POST /__hatchable/secrets/{key} — platform-validated write to the
  // right tier. Owner / user determined by which session the platform
  // sees on the request (account session for owner-tier, app-user
  // session for user-tier). Caller doesn't pick.
  async function save(key, value, opts = {}) {
    if (!key || typeof key !== 'string') {
      throw new Error('hatchable.config.save: key is required.');
    }
    const provider = opts.provider || null;
    const r = await fetch(`/__hatchable/secrets/${encodeURIComponent(key)}`, {
      method: 'POST',
      credentials: 'include',
      headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
      body: JSON.stringify({ value, provider }),
    });
    let data = null;
    try { data = await r.json(); } catch { /* */ }
    if (!r.ok) {
      const e = new Error((data && data.error) || `hatchable.config.save: ${r.status}`);
      e.status = r.status;
      e.body = data;
      throw e;
    }
    return data;
  }

  // Open the platform modal for a specific key. Returns a promise that
  // resolves when the user successfully saves, or rejects if they
  // dismiss. Useful when the agent wants to proactively prompt rather
  // than waiting for a SetupRequired catch.
  function prompt(key, opts = {}) {
    const scope = opts.scope || 'owner';
    const pasteable = scope === 'user' && opts.kind !== 'connection';
    let setupUrl = opts.setup_url || '/__hatchable/setup';
    if (scope === 'owner') setupUrl = withFrom(setupUrl);
    return requestModal({
      key, scope, setup_url: setupUrl, pasteable,
      label: opts.label || null,
      description: opts.description || null,
    }).then((r) => !!(r && r.saved));
  }

  // ---------------------------------------------------------------------
  // fetch interceptor — turns 412 setup_required into the modal flow
  // ---------------------------------------------------------------------

  const _fetch = window.fetch.bind(window);
  window.fetch = async function patchedFetch(input, init) {
    const response = await _fetch(input, init);

    // Cheap branch: only inspect non-2xx responses, and only ones that
    // claim JSON. Body can only be read once, so we clone first.
    if (response.status !== 412) return response;
    const ct = response.headers.get('content-type') || '';
    if (!ct.toLowerCase().includes('application/json')) return response;

    let payload = null;
    try { payload = await response.clone().json(); } catch { /* */ }
    if (!payload || payload.hint !== 'setup_required' || !payload.key) {
      return response;
    }

    const key = payload.key;
    const scope = payload.scope || 'owner';
    // Inline paste only works for a user-tier single-value secret — that
    // writes against the signed-in app user on THIS domain. Owner-tier
    // secrets are rejected here by design (the secrets endpoint 410s them;
    // they live in the console), and connections need an OAuth/console
    // flow — both of those explain + deep-link instead of pasting.
    const pasteable = scope === 'user' && payload.kind !== 'connection';
    const setupUrl = scope === 'owner'
      ? withFrom(payload.setup_url || '/__hatchable/setup')
      : (payload.setup_url || '/__hatchable/setup');

    // Silenced for this session → behave exactly as if un-patched.
    if (isDismissed(key)) return response;

    // One modal at a time; same-key 412s coalesce (see requestModal).
    let result;
    try {
      result = await requestModal({
        key, scope, setup_url: setupUrl, pasteable,
        label: payload.label || null,
        description: payload.description || null,
      });
    } catch { result = { saved: false }; }

    // Only a successful inline paste can transparently retry and hide the
    // 412 from the caller. Owner "Open setup →" navigates away; any
    // dismissal returns the original 412 so the caller degrades its way.
    if (!result || !result.saved) return response;
    try {
      return await _fetch(input, init);
    } catch {
      return response;
    }
  };

  // ---------------------------------------------------------------------
  // modal — minimal, neutral, styleable
  // ---------------------------------------------------------------------

  let modalRoot = null;

  function ensureRoot() {
    if (modalRoot) return modalRoot;
    modalRoot = document.createElement('div');
    modalRoot.className = 'hatchable-config-modal-root';
    modalRoot.style.cssText = 'position:fixed;inset:0;z-index:2147483646;display:none;';
    const styles = document.createElement('style');
    styles.textContent = `
      .hatchable-config-modal-root { font-family: -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; }
      .hatchable-config-modal-backdrop {
        position:absolute; inset:0; background:rgba(15,23,42,0.55);
        backdrop-filter: blur(4px); -webkit-backdrop-filter: blur(4px);
      }
      .hatchable-config-modal {
        position:absolute; top:50%; left:50%; transform:translate(-50%,-50%);
        width: min(440px, calc(100vw - 32px));
        background: #fff8e9; color: #252521;
        border:1px solid #ecdda7; border-radius:14px;
        box-shadow: 0 24px 48px rgba(0,0,0,0.18); padding:24px;
      }
      .hatchable-config-modal h3 {
        margin:0 0 6px; font-size:18px; font-weight:600; line-height:1.2;
      }
      .hatchable-config-modal p {
        margin:0 0 16px; font-size:14px; color:#6b6457; line-height:1.5;
      }
      .hatchable-config-modal code {
        font-family:"JetBrains Mono",ui-monospace,monospace; font-size:12px;
        background:rgba(0,0,0,0.05); padding:1px 6px; border-radius:4px;
      }
      .hatchable-config-modal input[type=text], .hatchable-config-modal input[type=password] {
        width:100%; box-sizing:border-box; padding:10px 12px;
        border:1px solid #ecdda7; border-radius:8px; background:#fff;
        font-family:"JetBrains Mono",ui-monospace,monospace; font-size:13px;
      }
      .hatchable-config-modal input[type=text]:focus, .hatchable-config-modal input[type=password]:focus {
        outline:2px solid #385c47; outline-offset:1px; border-color:#385c47;
      }
      .hatchable-config-modal .hatchable-config-row {
        display:flex; gap:8px; margin-top:8px;
      }
      .hatchable-config-modal button {
        font: inherit; cursor:pointer;
        border-radius:8px; padding:9px 14px; font-size:13px; font-weight:500;
      }
      .hatchable-config-modal button.hatchable-config-primary {
        background:#385c47; color:#fdfaf3; border:1px solid #2c4a39;
      }
      .hatchable-config-modal button.hatchable-config-primary:disabled {
        opacity:0.5; cursor:not-allowed;
      }
      .hatchable-config-modal button.hatchable-config-secondary {
        background:transparent; color:#6b6457; border:1px solid transparent;
      }
      .hatchable-config-modal .hatchable-config-error {
        margin-top:10px; font-size:12px; color:#991b1b; line-height:1.4;
      }
      .hatchable-config-modal .hatchable-config-toggle {
        background:rgba(0,0,0,0.04); color:#6b6457; border:1px solid transparent;
        padding:9px 10px;
      }
      .hatchable-config-modal .hatchable-config-actions {
        display:flex; gap:8px; justify-content:flex-end; margin-top:16px; align-items:center;
      }
      .hatchable-config-modal .hatchable-config-link {
        color:#385c47; font-size:12px; margin-right:auto; text-decoration:underline;
      }
      .hatchable-config-modal .hatchable-config-suppress {
        display:flex; align-items:center; gap:6px; margin-top:14px;
        font-size:12px; color:#6b6457; cursor:pointer; user-select:none;
      }
      .hatchable-config-modal .hatchable-config-suppress input { width:auto; margin:0; }
      .hatchable-config-modal a.hatchable-config-primary-link {
        background:#385c47; color:#fdfaf3; border:1px solid #2c4a39;
        border-radius:8px; padding:9px 14px; font-size:13px; font-weight:500;
        text-decoration:none; display:inline-block;
      }
    `;
    document.head.appendChild(styles);
    document.body.appendChild(modalRoot);
    return modalRoot;
  }

  // Render the SetupRequired dialog. Two shapes:
  //   - pasteable (user-tier secret): inline paste → save → caller retries.
  //   - link-only (owner-tier secret/AI key, or any connection): the value
  //     can't be written from this domain, so explain + deep-link to where
  //     it can. "Open setup →" is a real anchor; the browser navigates and
  //     the owner round-trips back via the ?from= param.
  // Resolves { saved, dismissed } — `dismissed` records a checked
  // "don't show again this session".
  function openModal({ key, scope, setup_url, label, description, pasteable }) {
    return new Promise((resolve) => {
      const root = ensureRoot();
      root.style.display = 'block';
      root.innerHTML = '';

      const backdrop = document.createElement('div');
      backdrop.className = 'hatchable-config-modal-backdrop';
      const modal = document.createElement('div');
      modal.className = 'hatchable-config-modal';

      const named = `<code>${escapeHtml(label || key)}</code>`;
      // Pasteable lead is followed by a paste instruction; the link-only
      // body stands alone (no redundant tail when a description already
      // explains the impact).
      const lead = description ? escapeHtml(description) : `This app needs ${named}.`;
      const linkBody = description
        ? escapeHtml(description)
        : `This app needs ${named} set up before this part of the app will work.`;
      const suppressRow =
        `<label class="hatchable-config-suppress"><input type="checkbox" /> Don't show this again this session</label>`;

      if (pasteable) {
        modal.innerHTML = `
          <h3>One thing left to configure</h3>
          <p>${lead} Paste the value below to continue.</p>
          <div class="hatchable-config-row">
            <input class="hatchable-config-value" type="password" autocomplete="off" spellcheck="false" placeholder="Paste value…" />
            <button type="button" class="hatchable-config-toggle">show</button>
          </div>
          <div class="hatchable-config-error" hidden></div>
          ${suppressRow}
          <div class="hatchable-config-actions">
            <a class="hatchable-config-link" href="${escapeHtml(setup_url)}" target="_blank" rel="noopener">Open full setup →</a>
            <button type="button" class="hatchable-config-secondary">Not now</button>
            <button type="button" class="hatchable-config-primary" disabled>Save</button>
          </div>
        `;
      } else {
        modal.innerHTML = `
          <h3>Setup needed</h3>
          <p>${linkBody}</p>
          ${suppressRow}
          <div class="hatchable-config-actions">
            <button type="button" class="hatchable-config-secondary">Not now</button>
            <a class="hatchable-config-primary-link" href="${escapeHtml(setup_url)}">Open setup →</a>
          </div>
        `;
      }
      root.append(backdrop, modal);

      const suppress = modal.querySelector('.hatchable-config-suppress input');
      const skipBtn = modal.querySelector('.hatchable-config-secondary');

      function finish(result) {
        const dismissed = !!(suppress && suppress.checked);
        if (dismissed) markDismissed(key);
        root.style.display = 'none';
        root.innerHTML = '';
        resolve({ saved: false, dismissed, ...result });
      }

      backdrop.addEventListener('click', () => finish({}));
      skipBtn.addEventListener('click', () => finish({}));
      modal.addEventListener('keydown', (e) => { if (e.key === 'Escape') finish({}); });

      // Link-only: record a checked suppression before the page unloads,
      // then let the anchor navigate normally. No pasteable wiring below.
      const openLink = modal.querySelector('.hatchable-config-primary-link');
      if (openLink) {
        openLink.addEventListener('click', () => {
          if (suppress && suppress.checked) markDismissed(key);
        });
        return;
      }

      // Pasteable wiring.
      const input = modal.querySelector('.hatchable-config-value');
      const toggle = modal.querySelector('.hatchable-config-toggle');
      const errEl = modal.querySelector('.hatchable-config-error');
      const saveBtn = modal.querySelector('.hatchable-config-primary');

      input.focus();
      input.addEventListener('input', () => { saveBtn.disabled = input.value.trim() === ''; });
      toggle.addEventListener('click', () => {
        const showing = input.type === 'text';
        input.type = showing ? 'password' : 'text';
        toggle.textContent = showing ? 'show' : 'hide';
        input.focus();
      });
      saveBtn.addEventListener('click', async () => {
        const value = input.value;
        if (!value) return;
        saveBtn.disabled = true;
        saveBtn.textContent = 'Saving…';
        errEl.hidden = true;
        try {
          await save(key, value);
          finish({ saved: true });
        } catch (err) {
          errEl.hidden = false;
          errEl.textContent = err.message || 'Save failed.';
          saveBtn.disabled = false;
          saveBtn.textContent = 'Save';
        }
      });
      input.addEventListener('keydown', (e) => { if (e.key === 'Enter' && !saveBtn.disabled) saveBtn.click(); });
    });
  }

  function escapeHtml(s) {
    return String(s).replace(/[&<>"']/g, (c) => (
      { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]
    ));
  }

  window.hatchable.config = { save, prompt };

  // ---------------------------------------------------------------------
  // <hatchable-config-prompt key="…"> — inline embeddable paste form
  // ---------------------------------------------------------------------
  //
  // For agents that want the paste UX inline (in their own settings
  // page, an onboarding flow, etc.) without monkey-patching fetch.
  // Renders the same form the modal uses, scoped to a single key.
  // Same security envelope: input belongs to the platform's shadow
  // DOM, save POSTs to /__hatchable/secrets/{key} — the value never
  // sits in agent-controlled DOM beyond pass-through.
  //
  //   <hatchable-config-prompt key="ANTHROPIC_API_KEY"></hatchable-config-prompt>
  //
  // Attributes:
  //   key     (required) the manifest key
  //   label   (optional) override the default label
  //   on-save (optional) name of a window function called after save
  //
  // Fires:
  //   "hatchable-config-saved" when the value is successfully stored
  //   "hatchable-config-failed" when a save fails

  if (!customElements.get('hatchable-config-prompt')) {
    class HatchableConfigPrompt extends HTMLElement {
      connectedCallback() {
        const key = (this.getAttribute('key') || '').trim();
        const label = this.getAttribute('label') || `Set ${key}`;
        if (!key) {
          this.innerHTML = '<div style="color:#991b1b;font-size:13px;">hatchable-config-prompt: missing key attribute</div>';
          return;
        }
        const shadow = this.attachShadow({ mode: 'open' });
        shadow.innerHTML = `
          <style>
            :host { display:block; font-family: -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; }
            .wrap { padding:16px; border:1px solid #ecdda7; border-radius:12px; background:#fff8e9; }
            label { font-size:12px; font-weight:600; color:#252521; display:block; margin-bottom:6px; }
            .row { display:flex; gap:8px; }
            input { flex:1; padding:10px 12px; border:1px solid #ecdda7; border-radius:8px; background:#fff; font-family:"JetBrains Mono",ui-monospace,monospace; font-size:13px; box-sizing:border-box; }
            button { font: inherit; cursor:pointer; border-radius:8px; padding:9px 14px; font-size:13px; font-weight:500; }
            button.primary { background:#385c47; color:#fdfaf3; border:1px solid #2c4a39; }
            button.primary:disabled { opacity:0.5; cursor:not-allowed; }
            button.toggle { background:rgba(0,0,0,0.04); color:#6b6457; border:1px solid transparent; padding:9px 10px; }
            .err { margin-top:8px; font-size:12px; color:#991b1b; }
            .ok { margin-top:8px; font-size:12px; color:#166534; }
          </style>
          <div class="wrap">
            <label>${escapeHtml(label)}</label>
            <div class="row">
              <input type="password" autocomplete="off" spellcheck="false" placeholder="Paste value…" />
              <button class="toggle" type="button">show</button>
              <button class="primary" type="button" disabled>Save</button>
            </div>
            <div class="err" hidden></div>
            <div class="ok" hidden>Saved.</div>
          </div>
        `;
        const input = shadow.querySelector('input');
        const toggle = shadow.querySelector('.toggle');
        const saveBtn = shadow.querySelector('.primary');
        const errEl = shadow.querySelector('.err');
        const okEl = shadow.querySelector('.ok');

        input.addEventListener('input', () => {
          saveBtn.disabled = input.value.trim() === '';
          errEl.hidden = true; okEl.hidden = true;
        });
        toggle.addEventListener('click', () => {
          const showing = input.type === 'text';
          input.type = showing ? 'password' : 'text';
          toggle.textContent = showing ? 'show' : 'hide';
          input.focus();
        });
        saveBtn.addEventListener('click', async () => {
          if (!input.value) return;
          saveBtn.disabled = true; saveBtn.textContent = 'Saving…';
          try {
            await save(key, input.value);
            okEl.hidden = false; errEl.hidden = true;
            input.value = '';
            saveBtn.textContent = 'Save';
            this.dispatchEvent(new CustomEvent('hatchable-config-saved', { bubbles: true, detail: { key } }));
            const onSave = this.getAttribute('on-save');
            if (onSave && typeof window[onSave] === 'function') window[onSave]({ key });
          } catch (err) {
            errEl.textContent = err.message || 'Save failed.';
            errEl.hidden = false; okEl.hidden = true;
            saveBtn.textContent = 'Save'; saveBtn.disabled = false;
            this.dispatchEvent(new CustomEvent('hatchable-config-failed', { bubbles: true, detail: { key, error: err } }));
          }
        });
        input.addEventListener('keydown', (e) => {
          if (e.key === 'Enter' && !saveBtn.disabled) saveBtn.click();
        });
      }
    }
    customElements.define('hatchable-config-prompt', HatchableConfigPrompt);
  }
})();
