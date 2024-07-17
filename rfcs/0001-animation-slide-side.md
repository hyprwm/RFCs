---
feature: changing slide side animation
start-date: 2024-06-16
author: Ciflire
---

# Summary

This RFC aims to agree on a rule for forced side option of the slide animation
style

# Motivation

The way the animation `slide` is used feels inconsistent in two ways at the
moment i'm writing this:

- First, it describes a side that can be the side by which the animation starts
  as well as its end. For example the keyword `bottom` used for animation
  `windowsIn` will move the window **from** the bottom to the desired position
  and for the animation `windowsOut` the window will be slided to the bottom of
  the screen
- Secondly, following that logic, the keyword `bottom` would be used to make the
  window pop from the bottom and close sliding to the bottom. The same keyword
  is then used for two different visual results.
- The use of the term "slide" makes the user consider the movement and not the
  side This RFC will allow to write the documentation as clear as possible
  without having to detail each case, or by try and fail by the user

# Detailed design

The design at the moment is that the orientation of the animation depends on
wether the window is closed or not.

```cpp
// src/managers/AnimationManager.cpp
void CAnimationManager::animationSlide(PHLWINDOW pWindow, std::string force, bool close) {
    pWindow->m_vRealSize.warp(false); // size we preserve in slide

    const auto GOALPOS  = pWindow->m_vRealPosition.goal();
    const auto GOALSIZE = pWindow->m_vRealSize.goal();

    const auto PMONITOR = g_pCompositor->getMonitorFromID(pWindow->m_iMonitorID);

    if (!PMONITOR)
        return; // unsafe state most likely

    Vector2D posOffset;

    if (force != "") {
        if (force == "bottom")
            posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y + PMONITOR->vecSize.y);
        else if (force == "left")
            posOffset = GOALPOS - Vector2D(GOALSIZE.x, 0.0);
        else if (force == "right")
            posOffset = GOALPOS + Vector2D(GOALSIZE.x, 0.0);
        else
            posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y - GOALSIZE.y);

        if (!close)
            pWindow->m_vRealPosition.setValue(posOffset);
        else
            pWindow->m_vRealPosition = posOffset;

        return;
    }

    const auto MIDPOINT = GOALPOS + GOALSIZE / 2.f;

    // check sides it touches
    const bool DISPLAYLEFT   = STICKS(pWindow->m_vPosition.x, PMONITOR->vecPosition.x + PMONITOR->vecReservedTopLeft.x);
    const bool DISPLAYRIGHT  = STICKS(pWindow->m_vPosition.x + pWindow->m_vSize.x, PMONITOR->vecPosition.x + PMONITOR->vecSize.x - PMONITOR->vecReservedBottomRight.x);
    const bool DISPLAYTOP    = STICKS(pWindow->m_vPosition.y, PMONITOR->vecPosition.y + PMONITOR->vecReservedTopLeft.y);
    const bool DISPLAYBOTTOM = STICKS(pWindow->m_vPosition.y + pWindow->m_vSize.y, PMONITOR->vecPosition.y + PMONITOR->vecSize.y - PMONITOR->vecReservedBottomRight.y);

    if (DISPLAYBOTTOM && DISPLAYTOP) {
        if (DISPLAYLEFT && DISPLAYRIGHT) {
            posOffset = GOALPOS + Vector2D(0.0, GOALSIZE.y);
        } else if (DISPLAYLEFT) {
            posOffset = GOALPOS - Vector2D(GOALSIZE.x, 0.0);
        } else {
            posOffset = GOALPOS + Vector2D(GOALSIZE.x, 0.0);
        }
    } else if (DISPLAYTOP) {
        posOffset = GOALPOS - Vector2D(0.0, GOALSIZE.y);
    } else if (DISPLAYBOTTOM) {
        posOffset = GOALPOS + Vector2D(0.0, GOALSIZE.y);
    } else {
        if (MIDPOINT.y > PMONITOR->vecPosition.y + PMONITOR->vecSize.y / 2.f)
            posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y + PMONITOR->vecSize.y);
        else
            posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y - GOALSIZE.y);
    }

    if (!close)
        pWindow->m_vRealPosition.setValue(posOffset);
    else
        pWindow->m_vRealPosition = posOffset;
}
```

I'm giving my example that i tested and apologize for the bad quality. I only
changed the `if` statements where `force` argument is not empty.

```cpp
if (force != "") {
    if (force == "bottom" && !close)
        posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y - PMONITOR->vecSize.y);
    else if (force == "bottom")
        posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y + PMONITOR->vecSize.y);
    else if (force == "left" && !close)
        posOffset = GOALPOS + Vector2D(GOALSIZE.x, 0.0);
    else if (force == "left")
        posOffset = GOALPOS - Vector2D(GOALSIZE.x, 0.0);
    else if (force == "right" && !close)
        posOffset = GOALPOS - Vector2D(GOALSIZE.x, 0.0);
    else if (force == "right")
        posOffset = GOALPOS + Vector2D(GOALSIZE.x, 0.0);
    else if (force == "top" && !close)
        posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y + PMONITOR->vecSize.y);
    else if (force == "top")
        posOffset = Vector2D(GOALPOS.x, PMONITOR->vecPosition.y - PMONITOR->vecSize.y);
```

I guess it could be refactored but i do not have the experience yet to do it
(intoducing a close bit for example). The slide direction does not depend on
whether the window is opened or closed. Meaning `slide top` will allways slide
the window to the top.

# Examples and Interactions

# Drawbacks

- Users will have to modify their configuration based on that change
- Could feel counterintuitive to users (as much as the actual behaviour is to
  me)

# Alternatives

- Keeping the way it is

# Prior art

[This issue](https://github.com/hyprwm/Hyprland/issues/6826) mentions the way
animation work. I will emphasize the end of the description section "using
`slide top` (it then slides down)". The user has to describe the exact visual
result of that animation. I will allow myself to infer that the need to give
this information means that it doesn't feel intuitive to the user. This is also
a pretty recent change as it was implemented and merged the 15th of june. Before
that the option existed but was not parsed (ref:
[6512](https://github.com/hyprwm/Hyprland/issues/6512))

# Unresolved questions

# Future work
