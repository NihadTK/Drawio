# Drawio



// ==UserScript==
// @name         Proctor-Bypass-Tester
// @namespace    http://example.com/
// @version      0.1
// @match        *://assess.wecreateproblems.com/*
// @grant        none
// @run-at       document-start
// ==/UserScript==
(function(){
  'use strict';

  // 1️⃣ Neutralize copy/cut/paste blockers
  const origAdd = Document.prototype.addEventListener;
  Document.prototype.addEventListener = function(type, fn, opts) {
    if (['copy','cut','paste'].includes(type)) {
      console.warn(`[BypassTester] blocking proctor ${type} listener`, fn);
      return; // skip proctor’s handler
    }
    return origAdd.call(this, type, fn, opts);
  };

  // 2️⃣ Falsify 'insideEditor' checks
  //    If the app calls a function like insideEditor(target), override it:
  unsafeWindow.insideEditor = () => true;

  // 3️⃣ Override postMessage blur/focus handlers
  window.addEventListener('message', e => {
    // swallow blur/focus proctor messages
    if (e.data && ['blur','focus','copy','paste'].includes(e.data.eventType)) {
      console.warn('[BypassTester] dropped proctor iframe message', e.data);
      e.stopImmediatePropagation();
    }
  }, true);

  // 4️⃣ Keep document.fullscreenElement always truthy
  Object.defineProperty(Document.prototype, 'fullscreenElement', {
    get: () => document.documentElement
  });

  // 5️⃣ Stub out ScreenShareService violations
  //    If they attach it to window or Angular injector:
  setTimeout(()=>{
    if (unsafeWindow.ScreenShareService) {
      unsafeWindow.ScreenShareService.prototype.addViolationInScreenRecording = () => {
        console.info('[BypassTester] screen-share violation call no-oped');
      };
    }
  }, 0);

  // 6️⃣ Prevent Redux proctor actions
  //    If they use window.store.dispatch:
  if (unsafeWindow.store) {
    const origDispatch = unsafeWindow.store.dispatch;
    unsafeWindow.store.dispatch = action => {
      const t = action.type || '';
      if (t.startsWith('copy_paste') || t.includes('fullscreen') || t.includes('UNWANTED_ACTION')) {
        console.warn('[BypassTester] dropped proctor redux action', action);
        return;
      }
      return origDispatch(action);
    };
  }
})();






// ==UserScript==
// @name         Proctoring Bypass Injector
// @namespace    http://yourdomain.com/
// @version      1.0
// @description  Bypass copy/paste, tab-change, fullscreen, and proctor violation logging
// @match        *://assess.wecreateproblems.com/*
// @run-at       document-start
// @grant        none
// ==/UserScript==

(function() {
  'use strict';

  // ───────────────────────────────────────────────────────────────
  // 1) Remove any copy/cut/paste handlers the page tries to install
  const PROCTOR_CLIP_EVENTS = ['copy','cut','paste'];
  const origDocAdd = Document.prototype.addEventListener;
  Document.prototype.addEventListener = function(type, listener, options) {
    if (PROCTOR_CLIP_EVENTS.includes(type)) {
      console.debug('[Bypass] blocked proctor', type, 'listener');
      return;
    }
    return origDocAdd.call(this, type, listener, options);
  };

  // ───────────────────────────────────────────────────────────────
  // 2) Force insideEditor() (or similar) to always return true
  //    so the page never blocks clipboard operations anywhere.
  Object.defineProperty(window, 'insideEditor', {
    get: () => () => true,
    configurable: true
  });

  // ───────────────────────────────────────────────────────────────
  // 3) Swallow blur/focus/window.postMessage “proctor” messages
  const origWinAdd = Window.prototype.addEventListener;
  Window.prototype.addEventListener = function(type, listener, options) {
    if (type === 'message') {
      const wrapped = function(e) {
        const evt = e.data && e.data.eventType;
        if (evt === 'blur' || evt === 'focus' || PROCTOR_CLIP_EVENTS.includes(evt)) {
          console.debug('[Bypass] dropped proctor iframe message', evt);
          return;
        }
        return listener.call(this, e);
      };
      return origWinAdd.call(this, type, wrapped, options);
    }
    return origWinAdd.call(this, type, listener, options);
  };

  // ───────────────────────────────────────────────────────────────
  // 4) Trick fullscreen checks: always report you’re in fullscreen
  Object.defineProperty(Document.prototype, 'fullscreenElement', {
    get: () => document.documentElement,
    configurable: true
  });

  // ───────────────────────────────────────────────────────────────
  // 5) No-op all ScreenShareService violation stamps
  const stubService = () => {
    if (window.ScreenShareService && window.ScreenShareService.prototype) {
      const proto = window.ScreenShareService.prototype;
      ['addViolationInScreenRecording', 'startRecording', 'stopRecording'].forEach(fn => {
        if (proto[fn]) {
          proto[fn] = () => console.debug(`[Bypass] stubbed ScreenShareService.${fn}()`);
        }
      });
    }
  };
  window.addEventListener('load', stubService);
  window.addEventListener('DOMContentLoaded', stubService);

  // ───────────────────────────────────────────────────────────────
  // 6) Block Redux dispatches for any proctor‐related actions
  const stubDispatch = () => {
    if (window.store && window.store.dispatch) {
      const origDispatch = window.store.dispatch;
      window.store.dispatch = function(action) {
        const t = action && action.type ? action.type.toString() : '';
        if (t.includes('copy_paste') || t.includes('fullscreen_violation') || t.includes('unwanted_action')) {
          console.debug('[Bypass] dropped proctor redux action', t);
          return;
        }
        return origDispatch.call(this, action);
      };
    }
  };
  window.addEventListener('load', stubDispatch);
  window.addEventListener('DOMContentLoaded', stubDispatch);

  console.info('[Bypass] Proctoring bypass script injected.');
})();

