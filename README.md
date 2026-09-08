For a validation-driven disabled state like this, the core UX problem is: native disabled attributes kill hover/focus events and screen-reader access, so "tooltip on disabled item" doesn't actually work reliably unless you build around it deliberately.
UX pattern (recommended)
Don't rely on hover-only tooltips — they fail on touch devices and disabled elements often don't fire mouse events at all. Instead:
Keep the option focusable but not selectable — use aria-disabled="true" instead of the native disabled attribute. This keeps it in the tab order so keyboard/screen-reader users can reach it and hear the reason.
Show the reason as always-visible inline subtext under the option label (small, muted, italic) rather than purely on hover — e.g.:
Code
This works identically on desktop, touch, and for screen readers (via aria-describedby), and avoids hover-discoverability problems entirely.
Reserve a small info icon + Emerald Tooltip/Popover as a secondary affordance for cases where the reason is long — icon is tabindex="0", opens on focus/click/hover, not hover-only.
If the list mixes many enabled/disabled items, visually group them (enabled first, disabled items greyed with reason, optionally a subtle divider) rather than interleaving — reduces scanning effort when a user is hunting for what they can pick.
Data contract
Push the "why" down from the backend rather than deriving it client-side — validations live server-side (legal form, roles, etc.), so the API should return structured reasons, not just a boolean:
Ts
Vue + TS + Emerald sketch
Vue
<template>
  <erd-select v-model="selected" label="Process context:">
    <template #option="{ option }">
      <div :aria-disabled="option.disabled" :tabindex="option.disabled ? 0 : -1">
        <span>{{ option.label }}</span>
        <p v-if="option.disabled" class="option-reason" :id="`reason-${option.value}`">
          {{ t(option.disabledReasonKey!) }}
        </p>
      </div>
    </template>
  </erd-select>
</template>

If Emerald's Select/Listbox doesn't expose an #option slot, you'll need a custom listbox built on Emerald's low-level primitives (or wrap erd-list) — check the Emerald docs for Combobox/Listbox slot support before committing to native <select>, since native selects can't render rich per-option markup at all.
A few implementation notes:
Intercept @select/@update:modelValue to no-op (or show a toast/inline message) if a disabled option is somehow activated via keyboard Enter, since aria-disabled doesn't block interaction the way native disabled does.
Keep reason text short (you're already planning for that) — one line, no punctuation-heavy explanations; save detail for a tooltip/link to docs if needed.
Cache the disabled/reason computation per customer record load, not per dropdown open, since these depend on customer attributes that don't change mid-session.
