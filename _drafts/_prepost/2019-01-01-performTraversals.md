---
layout:     post
title:      ""
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

```java
private void performTraversals() {
    // cache mView since it is used so much below...
    final View host = mView;

    if (DBG) {
        System.out.println("======================================");
        System.out.println("performTraversals");
        host.debug();
    }

    if (host == null || !mAdded)
        return;

    mIsInTraversal = true;
    mWillDrawSoon = true;
    boolean windowSizeMayChange = false;
    boolean surfaceChanged = false;
    WindowManager.LayoutParams lp = mWindowAttributes;

    int desiredWindowWidth;
    int desiredWindowHeight;

    final int viewVisibility = getHostVisibility();
    final boolean viewVisibilityChanged = !mFirst
            && (mViewVisibility != viewVisibility || mNewSurfaceNeeded
            // Also check for possible double visibility update, which will make current
            // viewVisibility value equal to mViewVisibility and we may miss it.
            || mAppVisibilityChanged);
    mAppVisibilityChanged = false;
    final boolean viewUserVisibilityChanged = !mFirst &&
            ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));

    WindowManager.LayoutParams params = null;
    if (mWindowAttributesChanged) {
        mWindowAttributesChanged = false;
        surfaceChanged = true;
        params = lp;
    }
    CompatibilityInfo compatibilityInfo =
            mDisplay.getDisplayAdjustments().getCompatibilityInfo();
    if (compatibilityInfo.supportsScreen() == mLastInCompatMode) {
        params = lp;
        mFullRedrawNeeded = true;
        mLayoutRequested = true;
        if (mLastInCompatMode) {
            params.privateFlags &= ~WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            mLastInCompatMode = false;
        } else {
            params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
            mLastInCompatMode = true;
        }
    }

    mWindowAttributesChangesFlag = 0;

    Rect frame = mWinFrame;
    if (mFirst) {
        mFullRedrawNeeded = true;
        mLayoutRequested = true;

        final Configuration config = mContext.getResources().getConfiguration();
        if (shouldUseDisplaySize(lp)) {
            // NOTE -- system code, won't try to do compat mode.
            Point size = new Point();
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else {
            desiredWindowWidth = mWinFrame.width();
            desiredWindowHeight = mWinFrame.height();
        }

        // We used to use the following condition to choose 32 bits drawing caches:
        // PixelFormat.hasAlpha(lp.format) || lp.format == PixelFormat.RGBX_8888
        // However, windows are now always 32 bits by default, so choose 32 bits
        mAttachInfo.mUse32BitDrawingCache = true;
        mAttachInfo.mWindowVisibility = viewVisibility;
        mAttachInfo.mRecomputeGlobalAttributes = false;
        mLastConfigurationFromResources.setTo(config);
        mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
        // Set the layout direction if it has not been set before (inherit is the default)
        if (mViewLayoutDirectionInitial == View.LAYOUT_DIRECTION_INHERIT) {
            host.setLayoutDirection(config.getLayoutDirection());
        }
        host.dispatchAttachedToWindow(mAttachInfo, 0);
        mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
        dispatchApplyInsets(host);
    } else {
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            if (DEBUG_ORIENTATION) Log.v(mTag, "View " + host + " resized to: " + frame);
            mFullRedrawNeeded = true;
            mLayoutRequested = true;
            windowSizeMayChange = true;
        }
    }

    if (viewVisibilityChanged) {
        mAttachInfo.mWindowVisibility = viewVisibility;
        host.dispatchWindowVisibilityChanged(viewVisibility);
        if (viewUserVisibilityChanged) {
            host.dispatchVisibilityAggregated(viewVisibility == View.VISIBLE);
        }
        if (viewVisibility != View.VISIBLE || mNewSurfaceNeeded) {
            endDragResizing();
            destroyHardwareResources();
        }
        if (viewVisibility == View.GONE) {
            // After making a window gone, we will count it as being
            // shown for the first time the next time it gets focus.
            mHasHadWindowFocus = false;
        }
    }

    // Non-visible windows can't hold accessibility focus.
    if (mAttachInfo.mWindowVisibility != View.VISIBLE) {
        host.clearAccessibilityFocus();
    }

    // Execute enqueued actions on every traversal in case a detached view enqueued an action
    getRunQueue().executeActions(mAttachInfo.mHandler);

    boolean insetsChanged = false;

    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {

        final Resources res = mView.getContext().getResources();

        if (mFirst) {
            // make sure touch mode code executes by setting cached value
            // to opposite of the added touch mode.
            mAttachInfo.mInTouchMode = !mAddedTouchMode;
            ensureTouchModeLocally(mAddedTouchMode);
        } else {
            if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
                insetsChanged = true;
            }
            if (!mPendingContentInsets.equals(mAttachInfo.mContentInsets)) {
                insetsChanged = true;
            }
            if (!mPendingStableInsets.equals(mAttachInfo.mStableInsets)) {
                insetsChanged = true;
            }
            if (!mPendingDisplayCutout.equals(mAttachInfo.mDisplayCutout)) {
                insetsChanged = true;
            }
            if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
                mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
            }
            if (!mPendingOutsets.equals(mAttachInfo.mOutsets)) {
                insetsChanged = true;
            }
            if (mPendingAlwaysConsumeSystemBars != mAttachInfo.mAlwaysConsumeSystemBars) {
                insetsChanged = true;
            }
            if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                    || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                windowSizeMayChange = true;

                if (shouldUseDisplaySize(lp)) {
                    // NOTE -- system code, won't try to do compat mode.
                    Point size = new Point();
                    mDisplay.getRealSize(size);
                    desiredWindowWidth = size.x;
                    desiredWindowHeight = size.y;
                } else {
                    Configuration config = res.getConfiguration();
                    desiredWindowWidth = dipToPx(config.screenWidthDp);
                    desiredWindowHeight = dipToPx(config.screenHeightDp);
                }
            }
        }

        // Ask host how big it wants to be
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }

    if (collectViewAttributes()) {
        params = lp;
    }
    if (mAttachInfo.mForceReportNewAttributes) {
        mAttachInfo.mForceReportNewAttributes = false;
        params = lp;
    }

    if (mFirst || mAttachInfo.mViewVisibilityChanged) {
        mAttachInfo.mViewVisibilityChanged = false;
        int resizeMode = mSoftInputMode &
                WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST;
        // If we are in auto resize mode, then we need to determine
        // what mode to use now.
        if (resizeMode == WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
            final int N = mAttachInfo.mScrollContainers.size();
            for (int i=0; i<N; i++) {
                if (mAttachInfo.mScrollContainers.get(i).isShown()) {
                    resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
                }
            }
            if (resizeMode == 0) {
                resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN;
            }
            if ((lp.softInputMode &
                    WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) != resizeMode) {
                lp.softInputMode = (lp.softInputMode &
                        ~WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) |
                        resizeMode;
                params = lp;
            }
        }
    }

    if (params != null) {
        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            if (!PixelFormat.formatHasAlpha(params.format)) {
                params.format = PixelFormat.TRANSLUCENT;
            }
        }
        mAttachInfo.mOverscanRequested = (params.flags
                & WindowManager.LayoutParams.FLAG_LAYOUT_IN_OVERSCAN) != 0;
    }

    if (mApplyInsetsRequested) {
        mApplyInsetsRequested = false;
        mLastOverscanRequested = mAttachInfo.mOverscanRequested;
        dispatchApplyInsets(host);
        if (mLayoutRequested) {
            // Short-circuit catching a new layout request here, so
            // we don't need to go through two layout passes when things
            // change due to fitting system windows, which can happen a lot.
            windowSizeMayChange |= measureHierarchy(host, lp,
                    mView.getContext().getResources(),
                    desiredWindowWidth, desiredWindowHeight);
        }
    }

    if (layoutRequested) {
        // Clear this now, so that if anything requests a layout in the
        // rest of this function we will catch it and re-run a full
        // layout pass.
        mLayoutRequested = false;
    }

    boolean windowShouldResize = layoutRequested && windowSizeMayChange
        && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
            || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.width() < desiredWindowWidth && frame.width() != mWidth)
            || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                    frame.height() < desiredWindowHeight && frame.height() != mHeight));
    windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;

    // If the activity was just relaunched, it might have unfrozen the task bounds (while
    // relaunching), so we need to force a call into window manager to pick up the latest
    // bounds.
    windowShouldResize |= mActivityRelaunched;

    // Determine whether to compute insets.
    // If there are no inset listeners remaining then we may still need to compute
    // insets in case the old insets were non-empty and must be reset.
    final boolean computesInternalInsets =
            mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
            || mAttachInfo.mHasNonEmptyGivenInternalInsets;

    boolean insetsPending = false;
    int relayoutResult = 0;
    boolean updatedConfiguration = false;

    final int surfaceGenerationId = mSurface.getGenerationId();

    final boolean isViewVisible = viewVisibility == View.VISIBLE;
    final boolean windowRelayoutWasForced = mForceNextWindowRelayout;
    boolean surfaceSizeChanged = false;

    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        mForceNextWindowRelayout = false;

        if (isViewVisible) {
            // If this window is giving internal insets to the window
            // manager, and it is being added or changing its visibility,
            // then we want to first give the window manager "fake"
            // insets to cause it to effectively ignore the content of
            // the window during layout.  This avoids it briefly causing
            // other windows to resize/move based on the raw frame of the
            // window, waiting until we can finish laying out this window
            // and get back to the window manager with the ultimately
            // computed insets.
            insetsPending = computesInternalInsets && (mFirst || viewVisibilityChanged);
        }

        if (mSurfaceHolder != null) {
            mSurfaceHolder.mSurfaceLock.lock();
            mDrawingAllowed = true;
        }

        boolean hwInitialized = false;
        boolean contentInsetsChanged = false;
        boolean hadSurface = mSurface.isValid();

        try {
            if (mAttachInfo.mThreadedRenderer != null) {
                // relayoutWindow may decide to destroy mSurface. As that decision
                // happens in WindowManager service, we need to be defensive here
                // and stop using the surface in case it gets destroyed.
                if (mAttachInfo.mThreadedRenderer.pause()) {
                    // Animations were running so we need to push a frame
                    // to resume them
                    mDirty.set(0, 0, mWidth, mHeight);
                }
                mChoreographer.mFrameInfo.addFlags(FrameInfo.FLAG_WINDOW_LAYOUT_CHANGED);
            }
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);

            // If the pending {@link MergedConfiguration} handed back from
            // {@link #relayoutWindow} does not match the one last reported,
            // WindowManagerService has reported back a frame from a configuration not yet
            // handled by the client. In this case, we need to accept the configuration so we
            // do not lay out and draw with the wrong configuration.
            if (!mPendingMergedConfiguration.equals(mLastReportedMergedConfiguration)) {
                if (DEBUG_CONFIGURATION) Log.v(mTag, "Visible with new config: "
                        + mPendingMergedConfiguration.getMergedConfiguration());
                performConfigurationChange(mPendingMergedConfiguration, !mFirst,
                        INVALID_DISPLAY /* same display */);
                updatedConfiguration = true;
            }

            final boolean overscanInsetsChanged = !mPendingOverscanInsets.equals(
                    mAttachInfo.mOverscanInsets);
            contentInsetsChanged = !mPendingContentInsets.equals(
                    mAttachInfo.mContentInsets);
            final boolean visibleInsetsChanged = !mPendingVisibleInsets.equals(
                    mAttachInfo.mVisibleInsets);
            final boolean stableInsetsChanged = !mPendingStableInsets.equals(
                    mAttachInfo.mStableInsets);
            final boolean cutoutChanged = !mPendingDisplayCutout.equals(
                    mAttachInfo.mDisplayCutout);
            final boolean outsetsChanged = !mPendingOutsets.equals(mAttachInfo.mOutsets);
            surfaceSizeChanged = (relayoutResult
                    & WindowManagerGlobal.RELAYOUT_RES_SURFACE_RESIZED) != 0;
            surfaceChanged |= surfaceSizeChanged;
            final boolean alwaysConsumeSystemBarsChanged =
                    mPendingAlwaysConsumeSystemBars != mAttachInfo.mAlwaysConsumeSystemBars;
            final boolean colorModeChanged = hasColorModeChanged(lp.getColorMode());
            if (contentInsetsChanged) {
                mAttachInfo.mContentInsets.set(mPendingContentInsets);
            }
            if (overscanInsetsChanged) {
                mAttachInfo.mOverscanInsets.set(mPendingOverscanInsets);             
                // Need to relayout with content insets.
                contentInsetsChanged = true;
            }
            if (stableInsetsChanged) {
                mAttachInfo.mStableInsets.set(mPendingStableInsets);
                // Need to relayout with content insets.
                contentInsetsChanged = true;
            }
            if (cutoutChanged) {
                mAttachInfo.mDisplayCutout.set(mPendingDisplayCutout);
                // Need to relayout with content insets.
                contentInsetsChanged = true;
            }
            if (alwaysConsumeSystemBarsChanged) {
                mAttachInfo.mAlwaysConsumeSystemBars = mPendingAlwaysConsumeSystemBars;
                contentInsetsChanged = true;
            }
            if (contentInsetsChanged || mLastSystemUiVisibility !=
                    mAttachInfo.mSystemUiVisibility || mApplyInsetsRequested
                    || mLastOverscanRequested != mAttachInfo.mOverscanRequested
                    || outsetsChanged) {
                mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
                mLastOverscanRequested = mAttachInfo.mOverscanRequested;
                mAttachInfo.mOutsets.set(mPendingOutsets);
                mApplyInsetsRequested = false;
                dispatchApplyInsets(host);
                // We applied insets so force contentInsetsChanged to ensure the
                // hierarchy is measured below.
                contentInsetsChanged = true;
            }
            if (visibleInsetsChanged) {
                mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
            }
            if (colorModeChanged && mAttachInfo.mThreadedRenderer != null) {
                mAttachInfo.mThreadedRenderer.setWideGamut(
                        lp.getColorMode() == ActivityInfo.COLOR_MODE_WIDE_COLOR_GAMUT);
            }

            if (!hadSurface) {
                if (mSurface.isValid()) {
                    // If we are creating a new surface, then we need to
                    // completely redraw it.
                    mFullRedrawNeeded = true;
                    mPreviousTransparentRegion.setEmpty();

                    // Only initialize up-front if transparent regions are not
                    // requested, otherwise defer to see if the entire window
                    // will be transparent
                    if (mAttachInfo.mThreadedRenderer != null) {
                        try {
                            hwInitialized = mAttachInfo.mThreadedRenderer.initialize(
                                    mSurface);
                            if (hwInitialized && (host.mPrivateFlags
                                    & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) == 0) {
                                // Don't pre-allocate if transparent regions
                                // are requested as they may not be needed
                                mAttachInfo.mThreadedRenderer.allocateBuffers();
                            }
                        } catch (OutOfResourcesException e) {
                            handleOutOfResourcesException(e);
                            return;
                        }
                    }
                }
            } else if (!mSurface.isValid()) {
                // If the surface has been removed, then reset the scroll
                // positions.
                if (mLastScrolledFocus != null) {
                    mLastScrolledFocus.clear();
                }
                mScrollY = mCurScrollY = 0;
                if (mView instanceof RootViewSurfaceTaker) {
                    ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
                }
                if (mScroller != null) {
                    mScroller.abortAnimation();
                }
                // Our surface is gone
                if (mAttachInfo.mThreadedRenderer != null &&
                        mAttachInfo.mThreadedRenderer.isEnabled()) {
                    mAttachInfo.mThreadedRenderer.destroy();
                }
            } else if ((surfaceGenerationId != mSurface.getGenerationId()
                    || surfaceSizeChanged || windowRelayoutWasForced || colorModeChanged)
                    && mSurfaceHolder == null
                    && mAttachInfo.mThreadedRenderer != null) {
                mFullRedrawNeeded = true;
                try {
                    // Need to do updateSurface (which leads to CanvasContext::setSurface and
                    // re-create the EGLSurface) if either the Surface changed (as indicated by
                    // generation id), or WindowManager changed the surface size. The latter is
                    // because on some chips, changing the consumer side's BufferQueue size may
                    // not take effect immediately unless we create a new EGLSurface.
                    // Note that frame size change doesn't always imply surface size change (eg.
                    // drag resizing uses fullscreen surface), need to check surfaceSizeChanged
                    // flag from WindowManager.
                    mAttachInfo.mThreadedRenderer.updateSurface(mSurface);
                } catch (OutOfResourcesException e) {
                    handleOutOfResourcesException(e);
                    return;
                }
            }

            final boolean freeformResizing = (relayoutResult
                    & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_FREEFORM) != 0;
            final boolean dockedResizing = (relayoutResult
                    & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_DOCKED) != 0;
            final boolean dragResizing = freeformResizing || dockedResizing;
            if (mDragResizing != dragResizing) {
                if (dragResizing) {
                    mResizeMode = freeformResizing
                            ? RESIZE_MODE_FREEFORM
                            : RESIZE_MODE_DOCKED_DIVIDER;
                    final boolean backdropSizeMatchesFrame =
                            mWinFrame.width() == mPendingBackDropFrame.width()
                                    && mWinFrame.height() == mPendingBackDropFrame.height();
                    // TODO: Need cutout?
                    startDragResizing(mPendingBackDropFrame, !backdropSizeMatchesFrame,
                            mPendingVisibleInsets, mPendingStableInsets, mResizeMode);
                } else {
                    // We shouldn't come here, but if we come we should end the resize.
                    endDragResizing();
                }
            }
            if (!mUseMTRenderer) {
                if (dragResizing) {
                    mCanvasOffsetX = mWinFrame.left;
                    mCanvasOffsetY = mWinFrame.top;
                } else {
                    mCanvasOffsetX = mCanvasOffsetY = 0;
                }
            }
        } catch (RemoteException e) {
        }

        if (DEBUG_ORIENTATION) Log.v(
                TAG, "Relayout returned: frame=" + frame + ", surface=" + mSurface);

        mAttachInfo.mWindowLeft = frame.left;
        mAttachInfo.mWindowTop = frame.top;

        // !!FIXME!! This next section handles the case where we did not get the
        // window size we asked for. We should avoid this by getting a maximum size from
        // the window session beforehand.
        if (mWidth != frame.width() || mHeight != frame.height()) {
            mWidth = frame.width();
            mHeight = frame.height();
        }

        if (mSurfaceHolder != null) {
            // The app owns the surface; tell it about what is going on.
            if (mSurface.isValid()) {
                // XXX .copyFrom() doesn't work!
                //mSurfaceHolder.mSurface.copyFrom(mSurface);
                mSurfaceHolder.mSurface = mSurface;
            }
            mSurfaceHolder.setSurfaceFrameSize(mWidth, mHeight);
            mSurfaceHolder.mSurfaceLock.unlock();
            if (mSurface.isValid()) {
                if (!hadSurface) {
                    mSurfaceHolder.ungetCallbacks();

                    mIsCreating = true;
                    SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
                    if (callbacks != null) {
                        for (SurfaceHolder.Callback c : callbacks) {
                            c.surfaceCreated(mSurfaceHolder);
                        }
                    }
                    surfaceChanged = true;
                }
                if (surfaceChanged || surfaceGenerationId != mSurface.getGenerationId()) {
                    SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
                    if (callbacks != null) {
                        for (SurfaceHolder.Callback c : callbacks) {
                            c.surfaceChanged(mSurfaceHolder, lp.format,
                                    mWidth, mHeight);
                        }
                    }
                }
                mIsCreating = false;
            } else if (hadSurface) {
                notifySurfaceDestroyed();
                mSurfaceHolder.mSurfaceLock.lock();
                try {
                    mSurfaceHolder.mSurface = new Surface();
                } finally {
                    mSurfaceHolder.mSurfaceLock.unlock();
                }
            }
        }

        final ThreadedRenderer threadedRenderer = mAttachInfo.mThreadedRenderer;
        if (threadedRenderer != null && threadedRenderer.isEnabled()) {
            if (hwInitialized
                    || mWidth != threadedRenderer.getWidth()
                    || mHeight != threadedRenderer.getHeight()
                    || mNeedsRendererSetup) {
                threadedRenderer.setup(mWidth, mHeight, mAttachInfo,
                        mWindowAttributes.surfaceInsets);
                mNeedsRendererSetup = false;
            }
        }

        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                 // Ask host how big it wants to be
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                // Implementation of weights from WindowManager.LayoutParams
                // We just grow the dimensions as needed and re-measure if
                // needs be
                int width = host.getMeasuredWidth();
                int height = host.getMeasuredHeight();
                boolean measureAgain = false;

                if (lp.horizontalWeight > 0.0f) {
                    width += (int) ((mWidth - width) * lp.horizontalWeight);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }
                if (lp.verticalWeight > 0.0f) {
                    height += (int) ((mHeight - height) * lp.verticalWeight);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                            MeasureSpec.EXACTLY);
                    measureAgain = true;
                }

                if (measureAgain) {         
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }

                layoutRequested = true;
            }
        }
    } else {
        // Not the first pass and no window/insets/visibility change but the window
        // may have moved and we need check that and if so to update the left and right
        // in the attach info. We translate only the window frame since on window move
        // the window manager tells us only for the new frame but the insets are the
        // same and we do not want to translate them more than once.
        maybeHandleWindowMove(frame);
    }

    if (surfaceSizeChanged) {
        updateBoundsSurface();
    }

    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    boolean triggerGlobalLayoutListener = didLayout
            || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);

        // By this point all views have been sized and positioned
        // We can compute the transparent area

        if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
            // start out transparent
            // TODO: AVOID THAT CALL BY CACHING THE RESULT?
            host.getLocationInWindow(mTmpLocation);
            mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                    mTmpLocation[0] + host.mRight - host.mLeft,
                    mTmpLocation[1] + host.mBottom - host.mTop);

            host.gatherTransparentRegion(mTransparentRegion);
            if (mTranslator != null) {
                mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
            }

            if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                mPreviousTransparentRegion.set(mTransparentRegion);
                mFullRedrawNeeded = true;
                // reconfigure window manager
                try {
                    mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                } catch (RemoteException e) {
                }
            }
        }
    }

    if (triggerGlobalLayoutListener) {
        mAttachInfo.mRecomputeGlobalAttributes = false;
        mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
    }

    if (computesInternalInsets) {
        // Clear the original insets.
        final ViewTreeObserver.InternalInsetsInfo insets = mAttachInfo.mGivenInternalInsets;
        insets.reset();

        // Compute new insets in place.
        mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
        mAttachInfo.mHasNonEmptyGivenInternalInsets = !insets.isEmpty();

        // Tell the window manager.
        if (insetsPending || !mLastGivenInsets.equals(insets)) {
            mLastGivenInsets.set(insets);

            // Translate insets to screen coordinates if needed.
            final Rect contentInsets;
            final Rect visibleInsets;
            final Region touchableRegion;
            if (mTranslator != null) {
                contentInsets = mTranslator.getTranslatedContentInsets(insets.contentInsets);
                visibleInsets = mTranslator.getTranslatedVisibleInsets(insets.visibleInsets);
                touchableRegion = mTranslator.getTranslatedTouchableArea(insets.touchableRegion);
            } else {
                contentInsets = insets.contentInsets;
                visibleInsets = insets.visibleInsets;
                touchableRegion = insets.touchableRegion;
            }

            try {
                mWindowSession.setInsets(mWindow, insets.mTouchableInsets,
                        contentInsets, visibleInsets, touchableRegion);
            } catch (RemoteException e) {
            }
        }
    }

    if (mFirst) {
        if (sAlwaysAssignFocus || !isInTouchMode()) {
            // handle first focus request
            if (DEBUG_INPUT_RESIZE) {
                Log.v(mTag, "First: mView.hasFocus()=" + mView.hasFocus());
            }
            if (mView != null) {
                if (!mView.hasFocus()) {
                    mView.restoreDefaultFocus();
                    if (DEBUG_INPUT_RESIZE) {
                        Log.v(mTag, "First: requested focused view=" + mView.findFocus());
                    }
                } else {
                    if (DEBUG_INPUT_RESIZE) {
                        Log.v(mTag, "First: existing focused view=" + mView.findFocus());
                    }
                }
            }
        } else {
            // Some views (like ScrollView) won't hand focus to descendants that aren't within
            // their viewport. Before layout, there's a good change these views are size 0
            // which means no children can get focus. After layout, this view now has size, but
            // is not guaranteed to hand-off focus to a focusable child (specifically, the edge-
            // case where the child has a size prior to layout and thus won't trigger
            // focusableViewAvailable).
            View focused = mView.findFocus();
            if (focused instanceof ViewGroup
                    && ((ViewGroup) focused).getDescendantFocusability()
                            == ViewGroup.FOCUS_AFTER_DESCENDANTS) {
                focused.restoreDefaultFocus();
            }
        }
    }

    final boolean changedVisibility = (viewVisibilityChanged || mFirst) && isViewVisible;
    final boolean hasWindowFocus = mAttachInfo.mHasWindowFocus && isViewVisible;
    final boolean regainedFocus = hasWindowFocus && mLostWindowFocus;
    if (regainedFocus) {
        mLostWindowFocus = false;
    } else if (!hasWindowFocus && mHadWindowFocus) {
        mLostWindowFocus = true;
    }

    if (changedVisibility || regainedFocus) {
        // Toasts are presented as notifications - don't present them as windows as well
        boolean isToast = (mWindowAttributes == null) ? false
                : (mWindowAttributes.type == WindowManager.LayoutParams.TYPE_TOAST);
        if (!isToast) {
            host.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED);
        }
    }

    mFirst = false;
    mWillDrawSoon = false;
    mNewSurfaceNeeded = false;
    mActivityRelaunched = false;
    mViewVisibility = viewVisibility;
    mHadWindowFocus = hasWindowFocus;

    if (hasWindowFocus && !isInLocalFocusMode()) {
        final boolean imTarget = WindowManager.LayoutParams
                .mayUseInputMethod(mWindowAttributes.flags);
        if (imTarget != mLastWasImTarget) {
            mLastWasImTarget = imTarget;
            InputMethodManager imm = mContext.getSystemService(InputMethodManager.class);
            if (imm != null && imTarget) {
                imm.onPreWindowFocus(mView, hasWindowFocus);
                imm.onPostWindowFocus(mView, mView.findFocus(),
                        mWindowAttributes.softInputMode,
                        !mHasHadWindowFocus, mWindowAttributes.flags);
            }
        }
    }

    // Remember if we must report the next draw.
    if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
        reportNextDraw();
    }

    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

    if (!cancelDraw) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }

        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }

    mIsInTraversal = false;
}
```

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```



```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    mLayoutRequested = false;
    mScrollMayChange = true;
    mInLayout = true;

    final View host = mView;
    if (host == null) {
        return;
    }
    if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
        Log.v(mTag, "Laying out " + host + " to (" +
                host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
    }

    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
    try {
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

        mInLayout = false;
        int numViewsRequestingLayout = mLayoutRequesters.size();
        if (numViewsRequestingLayout > 0) {
            // requestLayout() was called during layout.
            // If no layout-request flags are set on the requesting views, there is no problem.
            // If some requests are still pending, then we need to clear those flags and do
            // a full request/measure/layout pass to handle this situation.
            ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                    false);
            if (validLayoutRequesters != null) {
                // Set this flag to indicate that any further requests are happening during
                // the second pass, which may result in posting those requests to the next
                // frame instead
                mHandlingLayoutInLayoutRequest = true;

                // Process fresh layout requests, then measure and layout
                int numValidRequests = validLayoutRequesters.size();
                for (int i = 0; i < numValidRequests; ++i) {
                    final View view = validLayoutRequesters.get(i);
                    Log.w("View", "requestLayout() improperly called by " + view +
                            " during layout: running second layout pass");
                    view.requestLayout();
                }
                measureHierarchy(host, lp, mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);
                mInLayout = true;
                host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                mHandlingLayoutInLayoutRequest = false;

                // Check the valid requests again, this time without checking/clearing the
                // layout flags, since requests happening during the second pass get noop'd
                validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                if (validLayoutRequesters != null) {
                    final ArrayList<View> finalRequesters = validLayoutRequesters;
                    // Post second-pass requests to the next frame
                    getRunQueue().post(new Runnable() {
                        @Override
                        public void run() {
                            int numValidRequests = finalRequesters.size();
                            for (int i = 0; i < numValidRequests; ++i) {
                                final View view = finalRequesters.get(i);
                                Log.w("View", "requestLayout() improperly called by " + view +
                                        " during second layout pass: posting in next frame");
                                view.requestLayout();
                            }
                        }
                    });
                }
            }

        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    mInLayout = false;
}
```

```java
private void performDraw() {
    if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
        return;
    } else if (mView == null) {
        return;
    }

    final boolean fullRedrawNeeded = mFullRedrawNeeded || mReportNextDraw;
    mFullRedrawNeeded = false;

    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");

    boolean usingAsyncReport = false;
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        ArrayList<Runnable> commitCallbacks = mAttachInfo.mTreeObserver
                .captureFrameCommitCallbacks();
        if (mReportNextDraw) {
            usingAsyncReport = true;
            final Handler handler = mAttachInfo.mHandler;
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback((long frameNr) ->
                    handler.postAtFrontOfQueue(() -> {
                        // TODO: Use the frame number
                        pendingDrawFinished();
                        if (commitCallbacks != null) {
                            for (int i = 0; i < commitCallbacks.size(); i++) {
                                commitCallbacks.get(i).run();
                            }
                        }
                    }));
        } else if (commitCallbacks != null && commitCallbacks.size() > 0) {
            final Handler handler = mAttachInfo.mHandler;
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback((long frameNr) ->
                    handler.postAtFrontOfQueue(() -> {
                        for (int i = 0; i < commitCallbacks.size(); i++) {
                            commitCallbacks.get(i).run();
                        }
                    }));
        }
    }

    try {
        boolean canUseAsync = draw(fullRedrawNeeded);
        if (usingAsyncReport && !canUseAsync) {
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
            usingAsyncReport = false;
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

    // For whatever reason we didn't create a HardwareRenderer, end any
    // hardware animations that are now dangling
    if (mAttachInfo.mPendingAnimatingRenderNodes != null) {
        final int count = mAttachInfo.mPendingAnimatingRenderNodes.size();
        for (int i = 0; i < count; i++) {
            mAttachInfo.mPendingAnimatingRenderNodes.get(i).endAllAnimators();
        }
        mAttachInfo.mPendingAnimatingRenderNodes.clear();
    }

    if (mReportNextDraw) {
        mReportNextDraw = false;

        // if we're using multi-thread renderer, wait for the window frame draws
        if (mWindowDrawCountDown != null) {
            try {
                mWindowDrawCountDown.await();
            } catch (InterruptedException e) {
                Log.e(mTag, "Window redraw count down interrupted!");
            }
            mWindowDrawCountDown = null;
        }

        if (mAttachInfo.mThreadedRenderer != null) {
            mAttachInfo.mThreadedRenderer.setStopped(mStopped);
        }

        if (LOCAL_LOGV) {
            Log.v(mTag, "FINISHED DRAWING: " + mWindowAttributes.getTitle());
        }

        if (mSurfaceHolder != null && mSurface.isValid()) {
            SurfaceCallbackHelper sch = new SurfaceCallbackHelper(this::postDrawFinished);
            SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();

            sch.dispatchSurfaceRedrawNeededAsync(mSurfaceHolder, callbacks);
        } else if (!usingAsyncReport) {
            if (mAttachInfo.mThreadedRenderer != null) {
                mAttachInfo.mThreadedRenderer.fence();
            }
            pendingDrawFinished();
        }
    }
}
```
