---
title: Android WebView DrawFn 相关
date: 2020-11-16 13:05:17
tags:
---
在 chromium 的较高版本 webview 中，适配了android Q 的 vulkan 渲染。
在 webview 的历史实现中，由于和 chrome 架构差异，需要使用 DrawGL 相关方法在 android view 中绘制内容。

chrome 浏览器是完全独立的app，因此完全实现了自己的渲染，没有用android的 view 框架；webview 由于要作为一个 view 嵌入 app，所以必须依托于 android 的 view 对象，webview 通过反射调用 android.view.DisplayListCanvas 的 drawGLFunctor 方法实现绘制。
在最新的代码中，为了使用 android Q 的 vulkan 框架，webview 使用了新的实现方法用于调用vulkan。

## plat_support
plat_support 是个webview依赖的系统库，实现了webview的创建functor的方法，由于是平台相关，所以实在framework中提供。
https://cs.android.com/android/platform/superproject/+/master:frameworks/base/native/webview/plat_support/
在plat_support创建时会向一个webview的类中注册几个native方法，webview通过使用这些方法，创建系统版本对应的functor，如创建DrawGLFunctor的

```
const char kClassName[] = "com/android/webview/chromium/DrawGLFunctor";
const JNINativeMethod kJniMethods[] = {
    { "nativeCreateGLFunctor", "(J)J",
        reinterpret_cast<void*>(CreateGLFunctor) },
    { "nativeDestroyGLFunctor", "(J)V",
        reinterpret_cast<void*>(DestroyGLFunctor) },
    { "nativeSetChromiumAwDrawGLFunction", "(J)V",
        reinterpret_cast<void*>(SetChromiumAwDrawGLFunction) },
};
```
以及创建能使用vulkan的DrawFunctor的
```
char kClassName[]
const JNINativeMethod kJniMethods[] = {
    {"nativeGetFunctionTable", "()J",
     reinterpret_cast<void*>(GetDrawFnFunctionTable)},
};
```

在创建DrawFunctor的table中，看指向create_functor的函数指针，指向了真正的创建函数：
```
int CreateFunctor(void* data, AwDrawFnFunctorCallbacks* functor_callbacks) {
  static bool callbacks_initialized = false;
  static uirenderer::WebViewFunctorCallbacks webview_functor_callbacks = {
      .onSync = &onSync,
      .onContextDestroyed = &onContextDestroyed,
      .onDestroyed = &onDestroyed,
  };
  if (!callbacks_initialized) {
    switch (uirenderer::WebViewFunctor_queryPlatformRenderMode()) {
      case uirenderer::RenderMode::OpenGL_ES:
        webview_functor_callbacks.gles.draw = &draw_gl;
        break;
      case uirenderer::RenderMode::Vulkan:
        webview_functor_callbacks.vk.initialize = &initializeVk;
        webview_functor_callbacks.vk.draw = &drawVk;
        webview_functor_callbacks.vk.postDraw = &postDrawVk;
        break;
    }
    callbacks_initialized = true;
  }
  SupportData* support = new SupportData{
      .data = data,
      .callbacks = *functor_callbacks,
  };
  int functor = uirenderer::WebViewFunctor_create(
      support, webview_functor_callbacks,
      uirenderer::WebViewFunctor_queryPlatformRenderMode());
  if (functor <= 0) delete support;
  return functor;
}
```
首次初始化时会根据平台相关特性，选择使用opengl还是vulkan。

## WebView 调用DrawGLFunctor过程
这部分网上分析很多，不详细记录了，大概就是AwContents.onDraw的时候反射调用canvas里一个私有的方法。webview 通过周期地触发InProcessCommandBuffer::Flush方法，触发renderThread的合成。

需要注意的是，chrome和 webview 在渲染上最大的区别就是，command buffer service线程，在chrome上生产者和消费者都在同一个service线程，在webview上，生产者和消费者在不同的线程，因为webview的渲染不是chromium自己管理的，他只输出EGLImage，交给android系统的RenderThread处理 ConsumeTexture。于是，webview使用MailboxManagerSync，而chrome使用MailboxManagerImpl，webview使用的是加锁多线程共享的方式实现。
附生产和消费texture的调用栈：
```
(gdb) bt
#0  gpu::gles2::MailboxManagerSync::ConsumeTexture () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/mailbox_manager_sync.cc:194
#1  0xcbaa0f80 in gpu::gles2::GLES2DecoderImpl::DoCreateAndConsumeTextureINTERNAL () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/gles2_cmd_decoder.cc:18454
#2  0xcba85898 in gpu::gles2::GLES2DecoderImpl::HandleCreateAndConsumeTextureINTERNALImmediate () at ../../gpu/command_buffer/service/gles2_cmd_decoder_autogen.h:4946
#3  0xcba8f6a0 in gpu::gles2::GLES2DecoderImpl::DoCommandsImpl<false> () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/gles2_cmd_decoder.cc:5932
#4  0xcb9bdc54 in gpu::CommandBufferService::Flush () at ./../../gpu/command_buffer/service/command_buffer_service.cc:69
#5  0xcbb2163a in gpu::InProcessCommandBuffer::FlushOnGpuThread () at ../../gpu/ipc/in_process_command_buffer.cc:913
#6  0xca515508 in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#7  0xca52ffde in base::internal::FunctorTraits<void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), void>::Invoke<void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), base::WeakPtr<android_webview::AwCookieStoreWrapper>, base::OnceCallback<void ()> >(void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), base::WeakPtr<android_webview::AwCookieStoreWrapper>&&, base::OnceCallback<void ()>&&) () at ../../base/bind_internal.h:499
#8  base::internal::InvokeHelper<true, void>::MakeItSo<void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), base::WeakPtr<android_webview::AwCookieStoreWrapper>, base::OnceCallback<void ()> >(void (android_webview::AwCookieStoreWrapper::*&&)(base::OnceCallback<void ()>), base::WeakPtr<android_webview::AwCookieStoreWrapper>&&, base::OnceCallback<void ()>&&) ()
    at ../../base/bind_internal.h:619
#9  base::internal::Invoker<base::internal::BindState<void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), base::WeakPtr<android_webview::AwCookieStoreWrapper>, base::OnceCallback<void ()> >, void ()>::RunImpl<void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), std::__1::tuple<base::WeakPtr<android_webview::AwCookieStoreWrapper>, base::OnceCallback<void ()> >, 0u, 1u>(void (android_webview::AwCookieStoreWrapper::*&&)(base::OnceCallback<void ()>), std::__1::tuple<base::WeakPtr<android_webview::AwCookieStoreWrapper>, base::OnceCallback<void ()> >&&, std::__1::integer_sequence<unsigned int, 0u, 1u>) () at ../../base/bind_internal.h:672
#10 base::internal::Invoker<base::internal::BindState<void (android_webview::AwCookieStoreWrapper::*)(base::OnceCallback<void ()>), base::WeakPtr<android_webview::AwCookieStoreWrapper>, base::OnceCallback<void ()> >, void ()>::RunOnce(base::internal::BindStateBase*) () at ../../base/bind_internal.h:641
#11 0xca515508 in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#12 0xca5484bc in android_webview::TaskForwardingSequence::RunTask(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int) ()
    at ../../android_webview/browser/gfx/deferred_gpu_command_service.cc:108
#13 0xca548544 in base::internal::FunctorTraits<void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), void>::Invoke<void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), base::WeakPtr<android_webview::TaskForwardingSequence>, base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int>(void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), base::WeakPtr<android_webview::TaskForwardingSequence>&&, base::OnceCallback<void ()>&&, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >&&, unsigned int&&) () at ../../base/bind_internal.h:499
#14 base::internal::InvokeHelper<true, void>::MakeItSo<void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), base::WeakPtr<android_webview::TaskForwardingSequence>, base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int>(void (android_webview::TaskForwardingSequence::*&&)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), base::WeakPtr<android_webview::TaskForwardingSequence>&&, base::OnceCallback<void ()>&&, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >&&, unsigned int&&) () at ../../base/bind_internal.h:619
#15 base::internal::Invoker<base::internal::BindState<void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), base::WeakPtr<android_webview::TaskForwardingSequence>, base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int>, void ()>::RunImpl<void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), std::__1::tuple<base::WeakPtr<android_webview::TaskForwardingSequence>, base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int>, 0u, 1u, 2u, 3u>(void (android_webview::TaskForwardingSequence::*&&)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), std::__1::tuple<base::WeakPtr<android_webview::TaskForwardingSequence>, base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int>&&, std::__1::integer_sequence<unsigned int, 0u, 1u, 2u, 3u>) () at ../../base/bind_internal.h:672
#16 base::internal::Invoker<base::internal::BindState<void (android_webview::TaskForwardingSequence::*)(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int), base::WeakPtr<android_webview::TaskForwardingSequence>, base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >, unsigned int>, void ()>::RunOnce(base::internal::BindStateBase*) () at ../../base/bind_internal.h:641
#17 0xca515508 in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#18 0xca547f16 in android_webview::DeferredGpuCommandService::RunTasks () at ../../android_webview/browser/gfx/deferred_gpu_command_service.cc:243
#19 0xca547ec8 in android_webview::DeferredGpuCommandService::ScheduleTask(base::OnceCallback<void ()>, bool) () at ../../android_webview/browser/gfx/deferred_gpu_command_service.cc:188
#20 0xca5483ac in android_webview::TaskForwardingSequence::ScheduleTask(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >) ()
    at ../../android_webview/browser/gfx/deferred_gpu_command_service.cc:77
#21 0xcbb2156e in gpu::InProcessCommandBuffer::ScheduleGpuTask(base::OnceCallback<void ()>, std::__1::vector<gpu::SyncToken, std::__1::allocator<gpu::SyncToken> >) ()
    at ../../gpu/ipc/in_process_command_buffer.cc:849
#22 0xcbb219a6 in gpu::InProcessCommandBuffer::Flush () at ../../gpu/ipc/in_process_command_buffer.cc:977
#23 0xca7a9b84 in gpu::CommandBufferHelper::Flush () at ./../../gpu/command_buffer/client/cmd_buffer_helper.cc:182
#24 0xcb8b2800 in gpu::gles2::GLES2Implementation::FlushHelper () at ./../../gpu/command_buffer/client/gles2_implementation.cc:1388
#25 0xcb8b2a7c in gpu::gles2::GLES2Implementation::IssueShallowFlush () at ./../../gpu/command_buffer/client/gles2_implementation.cc:1378
#26 0xca549968 in android_webview::ParentOutputSurface::SwapBuffers () at ../../android_webview/browser/gfx/parent_output_surface.cc:52
#27 0xcba317dc in viz::GLRenderer::SwapBuffers () at ./../../components/viz/service/display/gl_renderer.cc:2901
#28 0xcba25d3a in viz::Display::DrawAndSwap () at ./../../components/viz/service/display/display.cc:523
#29 0xca54bba6 in android_webview::SurfacesInstance::DrawAndSwap () at ../../android_webview/browser/gfx/surfaces_instance.cc:239
#30 0xca548fc8 in android_webview::HardwareRenderer::Draw () at ../../android_webview/browser/gfx/hardware_renderer.cc:182
#31 0xca54a00c in android_webview::RenderThreadManager::DrawOnRT () at ../../android_webview/browser/gfx/render_thread_manager.cc:195
#32 0xca544be2 in android_webview::AwGLFunctor::DrawGL () at ../../android_webview/browser/gfx/aw_gl_functor.cc:119
---Type <return> to continue, or q <return> to quit---
#33 DrawGLFunction () at ../../android_webview/browser/gfx/aw_gl_functor.cc:27
#34 0xec21d328 in android::(anonymous namespace)::DrawGLFunctor::operator()(int, void*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libwebviewchromium_plat_support.so
#35 0xf26fc6de in android::uirenderer::skiapipeline::GLFunctorDrawable::onDraw(SkCanvas*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#36 0xf29b84b6 in SkDrawable::draw(SkCanvas*, SkMatrix const*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#37 0xf29b8ac2 in SkLiteDL::draw(SkCanvas*) const () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#38 0xf29a07e0 in android::uirenderer::skiapipeline::RenderNodeDrawable::drawContent(SkCanvas*) const () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#39 0xf29a0af6 in android::uirenderer::skiapipeline::RenderNodeDrawable::forceDraw(SkCanvas*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#40 0xf2703d3e in android::uirenderer::skiapipeline::SkiaPipeline::renderLayersImpl(android::uirenderer::LayerUpdateQueue const&, bool, bool) ()
   from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#41 0xf29d3d1a in android::uirenderer::skiapipeline::SkiaPipeline::renderFrame(android::uirenderer::LayerUpdateQueue const&, SkRect const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, bool, bool, android::uirenderer::Rect const&, sk_sp<SkSurface>) ()
   from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#42 0xf29d33de in android::uirenderer::skiapipeline::SkiaOpenGLPipeline::draw(android::uirenderer::renderthread::Frame const&, SkRect const&, SkRect const&, android::uirenderer::FrameBuilder::LightGeometry const&, android::uirenderer::LayerUpdateQueue*, android::uirenderer::Rect const&, bool, bool, android::uirenderer::BakedOpRenderer::LightInfo const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, android::uirenderer::FrameInfoVisualizer*) ()
   from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#43 0xf270c42c in android::uirenderer::renderthread::CanvasContext::draw() () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#44 0xf29d6b08 in std::__1::__function::__func<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0, std::__1::allocator<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0>, void ()>::operator() () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#45 0xf299fb30 in android::uirenderer::WorkQueue::process() () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#46 0xf271519e in android::uirenderer::renderthread::RenderThread::threadLoop() () from /tmp/adb-gdb-libs-803fd342/system/lib/libhwui.so
#47 0xf132f088 in android::Thread::_threadLoop(void*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libutils.so
#48 0xf1ea940a in __pthread_start(void*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libc.so
#49 0xf1e630ce in __start_thread () from /tmp/adb-gdb-libs-803fd342/system/lib/libc.so
```
```
(gdb) bt
#0  gpu::gles2::MailboxManagerSync::ProduceTexture () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/mailbox_manager_sync.cc:226
#1  0xcbae0318 in gpu::SharedImageBackingPassthroughGLTexture::ProduceLegacyMailbox ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:593
#2  0xcbad10b6 in gpu::SharedImageRepresentationFactoryRef::ProduceLegacyMailbox () at ../../gpu/command_buffer/service/shared_image_representation.h:73
#3  gpu::SharedImageFactory::RegisterBacking () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:287
#4  0xcbad0f96 in gpu::SharedImageFactory::CreateSharedImage () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:117
#5  0xcbb2bdf4 in gpu::SharedImageStub::OnCreateSharedImage () at ./../../gpu/ipc/service/shared_image_stub.cc:99
#6  0xcbb2baf4 in base::DispatchToMethodImpl<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>, 0u>(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&, std::__1::integer_sequence<unsigned int, 0u>) () at ../../base/tuple.h:52
#7  base::DispatchToMethod<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params> >(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&) () at ../../base/tuple.h:60
#8  IPC::DispatchToMethod<gpu::SharedImageStub, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), void, std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params> >(gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), void*, std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&) ()
    at ../../ipc/ipc_message_templates.h:51
#9  IPC::MessageT<GpuChannelMsg_CreateSharedImage_Meta, std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>, void>::Dispatch<gpu::SharedImageStub, gpu::SharedImageStub, void, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&)> () at ../../ipc/ipc_message_templates.h:146
#10 gpu::SharedImageStub::OnMessageReceived () at ./../../gpu/ipc/service/shared_image_stub.cc:69
#11 0xcbb280b8 in gpu::GpuChannel::HandleMessageHelper () at ./../../gpu/ipc/service/gpu_channel.cc:597
#12 0xcbb273b0 in gpu::GpuChannel::HandleMessage () at ./../../gpu/ipc/service/gpu_channel.cc:560
#13 0xcb9bf9b6 in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#14 gpu::Scheduler::RunNextTask () at ./../../gpu/command_buffer/service/scheduler.cc:527
#15 0xcb1088c8 in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#16 base::TaskAnnotator::RunTask () at ./../../base/task/common/task_annotator.cc:114
#17 base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoWorkImpl () at ./../../base/task/sequence_manager/thread_controller_with_message_pump_impl.cc:363
#18 0xcb10855c in base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoSomeWork () at ./../../base/task/sequence_manager/thread_controller_with_message_pump_impl.cc:214
#19 0xcb108d9c in non-virtual thunk to base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoSomeWork() () at ./../../base/time/time_now_posix.cc:52
#20 0xcb0e6e9c in base::MessagePumpDefault::Run () at ./../../base/message_loop/message_pump_default.cc:39
#21 0xcb1090fe in base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::Run () at ./../../base/task/sequence_manager/thread_controller_with_message_pump_impl.cc:448
#22 0xcb0f36de in base::RunLoop::RunWithTimeout () at ./../../base/run_loop.cc:161
#23 0xcb119ab2 in base::Thread::ThreadMain () at ./../../base/threading/thread.cc:312
#24 0xcb13b010 in base::(anonymous namespace)::ThreadFunc () at ./../../base/threading/platform_thread_posix.cc:81
#25 0xf1ea940a in __pthread_start(void*) () from /tmp/adb-gdb-libs-803fd342/system/lib/libc.so
#26 0xf1e630ce in __start_thread () from /tmp/adb-gdb-libs-803fd342/system/lib/libc.so
```
