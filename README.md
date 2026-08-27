// ==UserScript==
// @name         TFF Stripclub
// @namespace    http://tampermonkey.net/
// @version      6.0
// @description  Auto Sell Body + Auto Strip 5/5 — drawer rows removed safely
// @author       You
// @match        https://www.thefifthfamily.com/index.php*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    function refreshCounters() {
    updateButtons();
}


    // === BLOCK DRAWER CREATION COMPLETELY ===
(function() {
    const originalCreateElement = document.createElement;

    document.createElement = function(tag) {
        // If the game tries to create a drawer row, block it
        if (tag === 'div') {
            const el = originalCreateElement.call(document, tag);

            // When the element gets a class, intercept it
            Object.defineProperty(el, 'className', {
                set(v) {
                    if (v && v.includes('xp-drawer-row')) {
                        // Block drawer row creation
                        return;
                    }
                    this.setAttribute('class', v);
                },
                get() {
                    return this.getAttribute('class');
                }
            });

            return el;
        }

        return originalCreateElement.call(document, tag);
    };
})();





    // ====================== CONSTANTS ======================
    const SELECT_ID = 'xpCarSelect';
    const CHECK_EVERY = 200;
    const CONTROL_ID = 'xp-clean-controls';

    // ====================== UTILITIES ======================
    function randomDelay(min = 15, max = 20) {
        return min + Math.floor(Math.random() * (max - min + 1));
    }

    function getSelect() {
        return document.getElementById(SELECT_ID);
    }

    function isMonarchImperium(opt) {
        return opt && /2023 Monarch Imperium/i.test(opt.textContent);
    }

    function isFiveOfFive(opt) {
        return opt && /—\s*5\/5\b/.test(opt.textContent.trim());
    }

    function isZeroOfFive(opt) {
        return opt && /—\s*0\/5\b/.test(opt.textContent.trim());
    }

    function getCurrentOpt(select) {
        return select.selectedOptions[0] ||
               select.querySelector(`option[value="${select.value}"]`);
    }

    function forceSelect(opt) {
        const select = getSelect();
        if (!select || !opt) return;
        select.value = opt.value;
        Array.from(select.options).forEach(o => o.selected = false);
        opt.selected = true;
        select.dispatchEvent(new Event('input', { bubbles: true }));
        select.dispatchEvent(new Event('change', { bubbles: true }));
        if (document.activeElement === select) select.blur();
        console.log(`[XP] Switched → ${opt.value} | ${opt.textContent.trim()}`);
        lastAction = Date.now();
    }

    // ====================== CONFIRM BUTTON ======================
    function getConfirmButton() {
        const btn = document.getElementById('gc-confirm');
        if (!btn || btn.disabled) return null;
        return btn;
    }

function clickConfirmButton(label = 'Confirm') {
    const btn = getConfirmButton();
    if (!btn) return false;

    btn.click();
    console.log(`[XP] Clicked ${label}`);

    lastAction = Date.now();

    // Refresh counters after the game updates the select list
    setTimeout(refreshCounters, 120);

    return true;
}


    // ====================== STATE ======================
    let sellRunning = false;
    let stripRunning = false;
    let sellTimer = null;
    let stripTimer = null;
    let lastAction = 0;

    // ====================== AUTO SELL BODY ======================
    function getNextSellTarget(select) {
        return Array.from(select.querySelectorAll('option'))
            .filter(opt => opt.value && isMonarchImperium(opt))
            .sort((a, b) => parseInt(a.value, 10) - parseInt(b.value, 10))[0] || null;
    }

    function clickSellBodyButton() {
        const btn = document.getElementById('xpSellBody');
        if (btn && !btn.disabled) {
            btn.click();
            console.log('[XP Sell] Clicked Sell body');
            lastAction = Date.now();
            return true;
        }
        return false;
    }

    function sellTick() {
        if (!sellRunning) return;
        const select = getSelect();
        if (!select) return;

        const now = Date.now();
        const currentOpt = getCurrentOpt(select);

        if (clickConfirmButton('Sell confirm')) {
            lastAction = Date.now();
            return;
        }

        if (!currentOpt || !currentOpt.value || !isMonarchImperium(currentOpt)) {
            const next = getNextSellTarget(select);
            if (next) {
                forceSelect(next);
                console.log('[XP Sell] Selected next Monarch Imperium');
            } else {
                console.log('[XP Sell] No more 2023 Monarch Imperium left. Stopping.');
                stopSell();
            }
            return;
        }

        if (now - lastAction > 180) {
            if (!clickSellBodyButton()) {
                const next = getNextSellTarget(select);
                if (next && next.value !== currentOpt.value) {
                    forceSelect(next);
                }
            }
        }
    }

    function startSell() {
        if (sellRunning) return;
        stopStrip();
        sellRunning = true;
        lastAction = 0;
        updateButtons();
        console.log('[XP Sell] Started');
        sellTick();
        sellTimer = setInterval(sellTick, CHECK_EVERY);
    }

    function stopSell() {
        sellRunning = false;
        if (sellTimer) {
            clearInterval(sellTimer);
            sellTimer = null;
        }
        updateButtons();
        console.log('[XP Sell] Stopped');
    }

    // ====================== AUTO STRIP ======================
    function getNextStripTarget(select) {
        return Array.from(select.querySelectorAll('option'))
            .filter(opt => opt.value && isFiveOfFive(opt) && isMonarchImperium(opt))
            .sort((a, b) => parseInt(a.value, 10) - parseInt(b.value, 10))[0] || null;
    }

    function clickStripButton() {
        const sellBtn = document.getElementById('xpSellBody');
        const chopBtn = document.getElementById('xpChop');
        const btn = sellBtn || chopBtn;
        if (btn && !btn.disabled) {
            btn.click();
            console.log('[XP Strip] Clicked strip');
            lastAction = Date.now();
            return true;
        }
        return false;
    }

    function stripTick() {
        if (!stripRunning) return;
        const select = getSelect();
        if (!select) return;

        const now = Date.now();
        const currentOpt = getCurrentOpt(select);

        if (clickConfirmButton('Strip confirm')) return;

        if (!currentOpt || !currentOpt.value ||
            isZeroOfFive(currentOpt) ||
            !isMonarchImperium(currentOpt) ||
            !isFiveOfFive(currentOpt)) {
            const next = getNextStripTarget(select);
            if (next) {
                forceSelect(next);
            } else {
                console.log('[XP Strip] No more 5/5 left. Stopping.');
                stopStrip();
            }
            return;
        }

        if (isFiveOfFive(currentOpt) && isMonarchImperium(currentOpt) &&
            (now - lastAction > randomDelay(50, 100))) {
            clickStripButton();
        }
    }

    function startStrip() {
        if (stripRunning) return;
        stopSell();
        stripRunning = true;
        lastAction = 0;
        updateButtons();
        console.log('[XP Strip] Started');
        stripTick();
        stripTimer = setInterval(stripTick, CHECK_EVERY);
    }

    function stopStrip() {
        stripRunning = false;
        if (stripTimer) {
            clearInterval(stripTimer);
            stripTimer = null;
        }
        updateButtons();
        console.log('[XP Strip] Stopped');
    }

    // ====================== REMOVE DRAWER ROWS SAFELY ======================
    function nukeDrawerRows() {
        const rows = document.querySelectorAll('.xp-drawer-row');
        if (rows.length > 0) {
            rows.forEach(r => r.remove());
        }
    }

    // Run every 300ms — safe, does NOT touch your buttons
    setInterval(nukeDrawerRows, 300);

    // === COUNT REMAINING CARS ===
function countSellableBodies() {
    const select = getSelect();
    if (!select) return 0;

    return Array.from(select.options)
        .filter(opt => opt.value && isMonarchImperium(opt))
        .length;
}

function countStripTargets() {
    const select = getSelect();
    if (!select) return 0;

    return Array.from(select.options)
        .filter(opt => opt.value && isMonarchImperium(opt) && isFiveOfFive(opt))
        .length;
}

    // ====================== CONTROL UI ======================






function updateButtons() {
    const sellBtn = document.getElementById('xp-btn-sell');
    const stripBtn = document.getElementById('xp-btn-strip');

    const bodiesLeft = countSellableBodies();
    const stripLeft = countStripTargets();

    if (sellBtn) {
        sellBtn.textContent = sellRunning
            ? `⏳ Selling… (${bodiesLeft} left)`
            : `▶ Auto Sell Body (${bodiesLeft})`;
        sellBtn.style.background = sellRunning ? '#b45309' : '#0d9488';
    }

    if (stripBtn) {
        stripBtn.textContent = stripRunning
            ? `⏳ Stripping… (${stripLeft} left)`
            : `▶ Auto Strip 5/5 (${stripLeft})`;
        stripBtn.style.background = stripRunning ? '#b45309' : '#7c3aed';
    }
}



    function addControls() {
        if (document.getElementById(CONTROL_ID)) return;

        const bar = document.createElement('div');
        bar.id = CONTROL_ID;
        bar.style.cssText = `
            position: fixed;
            top: 120px;
            right: 12px;
            z-index: 999999;
            display: flex;
            flex-direction: column;
            gap: 8px;
            font-family: system-ui, sans-serif;
            user-select: none;
        `;

        const btnStyle = `
            padding: 9px 14px;
            color: white;
            border: none;
            border-radius: 6px;
            font-weight: 600;
            font-size: 13px;
            cursor: pointer;
            box-shadow: 0 3px 10px rgba(0,0,0,0.3);
            white-space: nowrap;
            min-width: 150px;
        `;

        const sellBtn = document.createElement('button');
        sellBtn.id = 'xp-btn-sell';
        sellBtn.style.cssText = btnStyle + 'background:#0d9488;';
        sellBtn.onclick = () => sellRunning ? stopSell() : startSell();

        const stripBtn = document.createElement('button');
        stripBtn.id = 'xp-btn-strip';
        stripBtn.style.cssText = btnStyle + 'background:#7c3aed;';
        stripBtn.onclick = () => stripRunning ? stopStrip() : startStrip();

        bar.append(sellBtn, stripBtn);
        document.body.appendChild(bar);

        updateButtons();
    }

    // ====================== INIT ======================

    setInterval(refreshCounters, 500);


    const finder = setInterval(() => {
        if (getSelect()) {
            clearInterval(finder);
            addControls();
            console.log('[XP Clean] Ready — Bulk removed, drawer nuked');
        }
    }, 400);

})();
