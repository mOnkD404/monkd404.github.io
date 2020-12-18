---
title: Android WebView DrawFn 相关
date: 2020-09-22 13:05:17
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
RasterTileWorker
```
#0  gpu::SharedImageInterfaceProxy::CreateSharedImage () at ../../gpu/ipc/client/shared_image_interface_proxy.cc:63
#1  0xc2ae3952 in cc::(anonymous namespace)::RasterizeSourceOOP () at ./../../cc/raster/gpu_raster_buffer_provider.cc:142
#2  cc::GpuRasterBufferProvider::PlaybackOnWorkerThreadInternal () at ./../../cc/raster/gpu_raster_buffer_provider.cc:546
#3  cc::GpuRasterBufferProvider::PlaybackOnWorkerThread () at ./../../cc/raster/gpu_raster_buffer_provider.cc:470
#4  cc::GpuRasterBufferProvider::RasterBufferImpl::Playback () at ./../../cc/raster/gpu_raster_buffer_provider.cc:325
#5  0xc2b4b6c8 in cc::(anonymous namespace)::RasterTaskImpl::RunOnWorkerThread () at ./../../cc/tiles/tile_manager.cc:160
#6  0xc365e4d2 in content::CategorizedWorkerPool::RunTaskInCategoryWithLockAcquired () at ./../../content/renderer/categorized_worker_pool.cc:400
#7  content::CategorizedWorkerPool::RunTaskWithLockAcquired () at ./../../content/renderer/categorized_worker_pool.cc:378
#8  content::CategorizedWorkerPool::Run () at ./../../content/renderer/categorized_worker_pool.cc:260
#9  content::(anonymous namespace)::CategorizedWorkerPoolThread::Run () at ./../../content/renderer/categorized_worker_pool.cc:55
#10 0xc21896a6 in base::SimpleThread::ThreadMain () at ./../../base/threading/simple_thread.cc:80
#11 0xc21ad234 in base::(anonymous namespace)::ThreadFunc () at ./../../base/threading/platform_thread_posix.cc:81
```
InProcGPUThread
```

#0  gpu::SharedImageBackingFactoryGLTexture::MakeBacking ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:1222
#1  0xc2d72960 in gpu::SharedImageBackingFactoryGLTexture::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:1052
#2  0xc2d72442 in gpu::SharedImageBackingFactoryGLTexture::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:857
#3  0xc2d74fe6 in gpu::SharedImageFactory::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:115
#4  0xc2dd1540 in gpu::SharedImageStub::OnCreateSharedImage () at ./../../gpu/ipc/service/shared_image_stub.cc:99
#5  0xc2dd1250 in base::DispatchToMethodImpl<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>, 0u>(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&, std::__1::integer_sequence<unsigned int, 0u>) ()
    at ../../base/tuple.h:52
#6  base::DispatchToMethod<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params> >(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&) () at ../../base/tuple.h:60
#7  IPC::DispatchToMethod<gpu::SharedImageStub, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), void, std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params> >(gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), void*, std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&) () at ../../ipc/ipc_message_templates.h:51
#8  IPC::MessageT<GpuChannelMsg_CreateSharedImage_Meta, std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>, void>::Dispatch<gpu::SharedImageStub, gpu::SharedImageStub, void, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&)> ()
    at ../../ipc/ipc_message_templates.h:146
#9  gpu::SharedImageStub::OnMessageReceived () at ./../../gpu/ipc/service/shared_image_stub.cc:69
#10 0xc2dcd8b0 in gpu::GpuChannel::HandleMessageHelper () at ./../../gpu/ipc/service/gpu_channel.cc:614
#11 0xc2dccbbc in gpu::GpuChannel::HandleMessage () at ./../../gpu/ipc/service/gpu_channel.cc:570
#12 0xc2c337ce in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#13 gpu::Scheduler::RunNextTask () at ./../../gpu/command_buffer/service/scheduler.cc:527
#14 0xc2245d46 in base::OnceCallback<void ()>::Run() && () at ../../base/callback.h:97
#15 base::TaskAnnotator::RunTask () at ./../../base/task/common/task_annotator.cc:114
#16 0xc22501f0 in base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoWorkImpl ()
    at ./../../base/task/sequence_manager/thread_controller_with_message_pump_impl.cc:363
#17 0xc224ff5a in base::sequence_manager::internal::ThreadControllerWithMessagePumpImpl::DoSomeWork ()
```

RenderThread 中 InProcessCommandBuffer建立
```
Thread 28 "RenderThread" hit Breakpoint 1, gpu::InProcessCommandBuffer::InProcessCommandBuffer ()
    at ../../gpu/ipc/in_process_command_buffer.cc:285
285     }
(gdb) bt
#0  gpu::InProcessCommandBuffer::InProcessCommandBuffer () at ../../gpu/ipc/in_process_command_buffer.cc:285
#1  0xc09722f2 in std::__1::make_unique<gpu::InProcessCommandBuffer, gpu::CommandBufferTaskExecutor*&, GURL>(gpu::CommandBufferTaskExecutor*&, GURL&&) () at ../../buildtools/third_party/libc++/trunk/include/memory:3131
#2  gpu::GLInProcessContext::Initialize () at ../../gpu/ipc/gl_in_process_context.cc:75
#3  0xbf09bf00 in android_webview::AwRenderThreadContextProvider::AwRenderThreadContextProvider ()
    at ../../android_webview/browser/gfx/aw_render_thread_context_provider.cc:69
#4  android_webview::AwRenderThreadContextProvider::Create () at ../../android_webview/browser/gfx/aw_render_thread_context_provider.cc:32
#5  0xbf0a25f4 in android_webview::SurfacesInstance::SurfacesInstance () at ../../android_webview/browser/gfx/surfaces_instance.cc:129
#6  android_webview::SurfacesInstance::GetOrCreateInstance () at ../../android_webview/browser/gfx/surfaces_instance.cc:55
#7  0xbf09f3e4 in android_webview::HardwareRenderer::HardwareRenderer () at ../../android_webview/browser/gfx/hardware_renderer.cc:33
#8  0xbf0a0f4c in android_webview::RenderThreadManager::DrawOnRT () at ../../android_webview/browser/gfx/render_thread_manager.cc:203
#9  0xbf09a31a in android_webview::AwDrawFnImpl::DrawInternal<AwDrawFn_DrawGLParams> ()
    at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:624
#10 android_webview::AwDrawFnImpl::DrawGL () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:275
#11 android_webview::(anonymous namespace)::DrawGLWrapper () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:133
#12 0xe0d68262 in android::(anonymous namespace)::draw_gl(int, void*, android::uirenderer::DrawGlInfo const&) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libwebviewchromium_plat_support.so
#13 0xebb53e20 in android::uirenderer::WebViewFunctor::drawGl(android::uirenderer::DrawGlInfo const&) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#14 0xebb5356e in android::uirenderer::skiapipeline::GLFunctorDrawable::onDraw(SkCanvas*) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#15 0xeba7a024 in SkCanvas::onDrawDrawable(SkDrawable*, SkMatrix const*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#16 0xeba725f4 in android::uirenderer::skiapipeline::RenderNodeDrawable::drawContent(SkCanvas*) const ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#17 0xeba98a42 in android::uirenderer::skiapipeline::RenderNodeDrawable::forceDraw(SkCanvas*) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#18 0xeba9dcc8 in android::uirenderer::skiapipeline::SkiaPipeline::renderLayersImpl(android::uirenderer::LayerUpdateQueue const&, bool) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#19 0xeba9be44 in android::uirenderer::skiapipeline::SkiaPipeline::renderFrame(android::uirenderer::LayerUpdateQueue const&, SkRect const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, bool, android::uirenderer::Rect const&, sk_sp<SkSurface>, SkMatrix const&) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
```
GPU线程 创建 SharedImageFactory
```
#0  gpu::SharedImageFactory::SharedImageFactory ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:96
#1  0xc0993164 in std::__1::make_unique<gpu::SharedImageFactory, gpu::GpuPreferences const&, gpu::GpuDriverBugWorkarounds const&, gpu::GpuFeatureInfo const&, gpu::SharedContextState*, gpu::MailboxManager*, gpu::SharedImageManager*, gpu::ImageFactory*, gpu::SharedImageStub*, bool>(gpu::GpuPreferences const&, gpu::GpuDriverBugWorkarounds const&, gpu::GpuFeatureInfo const&, gpu::SharedContextState*&&, gpu::MailboxManager*&&, gpu::SharedImageManager*&&, gpu::ImageFactory*&&, gpu::SharedImageStub*&&, bool&&) ()
    at ../../buildtools/third_party/libc++/trunk/include/memory:3131
#2  gpu::SharedImageStub::MakeContextCurrentAndCreateFactory () at ./../../gpu/ipc/service/shared_image_stub.cc:310
#3  0xc0990360 in gpu::SharedImageStub::Create () at ./../../gpu/ipc/service/shared_image_stub.cc:51
#4  gpu::GpuChannel::CreateSharedImageStub () at ./../../gpu/ipc/service/gpu_channel.cc:598
#5  gpu::GpuChannel::Create () at ./../../gpu/ipc/service/gpu_channel.cc:432
#6  gpu::GpuChannelManager::EstablishChannel () at ./../../gpu/ipc/service/gpu_channel_manager.cc:171
#7  0xbf485426 in viz::GpuServiceImpl::EstablishGpuChannel(int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>) () at ./../../components/viz/service/gl/gpu_service_impl.cc:648
#8  0xbf4887a4 in base::internal::FunctorTraits<void (viz::GpuServiceImpl::*)(int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>), void>::Invoke<void (viz::GpuServiceImpl::*)(int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>), base::WeakPtr<viz::GpuServiceImpl>, int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)> >(void (viz::GpuServiceImpl::*)(int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>), base::WeakPtr<viz::GpuServiceImpl>&&, int&&, unsigned long long&&, bool&&, bool&&, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>&&) () at ../../base/bind_internal.h:499
#9  base::internal::InvokeHelper<true, void>::MakeItSo<void (viz::GpuServiceImpl::*)(int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>), base::WeakPtr<viz::GpuServiceImpl>, int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)> >(void (viz::GpuServiceImpl::*&&)(int, unsigned long long, bool, bool, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>), base::WeakPtr<viz::GpuServiceImpl>&&, int&&, unsigned long long&&, bool&&, bool&&, base::OnceCallback<void (mojo::ScopedHandleBase<mojo::MessagePipeHandle>)>&&) () at ../../base/bind_internal.h:619
```
UI线程创建GPUchannel
```
#0  viz::GpuClient::PreEstablishGpuChannel () at ./../../components/viz/host/gpu_client.cc:66
#1  0xbf5d79dc in content::RenderProcessHostImpl::Init () at ./../../content/browser/renderer_host/render_process_host_impl.cc:1822
#2  0xbf53f5cc in content::RenderFrameHostManager::InitRenderView () at ./../../content/browser/frame_host/render_frame_host_manager.cc:2087
#3  0xbf53e7a4 in content::RenderFrameHostManager::ReinitializeRenderFrame ()
    at ./../../content/browser/frame_host/render_frame_host_manager.cc:2253
#4  0xbf53dc44 in content::RenderFrameHostManager::GetFrameHostForNavigation ()
    at ./../../content/browser/frame_host/render_frame_host_manager.cc:740
#5  0xbf53d9fc in content::RenderFrameHostManager::DidCreateNavigationRequest ()
    at ./../../content/browser/frame_host/render_frame_host_manager.cc:574
#6  0xbf5102b6 in content::FrameTreeNode::CreatedNavigationRequest () at ./../../content/browser/frame_host/frame_tree_node.cc:430
#7  0xbf5299d0 in content::NavigatorImpl::Navigate () at ./../../content/browser/frame_host/navigator_impl.cc:346
#8  0xbf51da24 in content::NavigationControllerImpl::NavigateWithoutEntry ()
    at ./../../content/browser/frame_host/navigation_controller_impl.cc:2878
#9  content::NavigationControllerImpl::LoadURLWithParams () at ./../../content/browser/frame_host/navigation_controller_impl.cc:973
#10 0xbf512bac in content::NavigationControllerAndroid::LoadUrl () at ./../../content/browser/frame_host/navigation_controller_android.cc:294
#11 Java_com_bytedance_org_chromium_content_browser_framehost_NavigationControllerImpl_nativeLoadUrl ()
    at gen/content/public/android/content_jni_headers/content/jni/NavigationControllerImpl_jni.h:241
```
GPU线程创建MailboxManagerSync
```
#0  gpu::gles2::CreateMailboxManager () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/mailbox_manager_factory.cc:17
#1  0xbf04696c in android_webview::DeferredGpuCommandService::CreateDeferredGpuCommandService ()
    at ../../android_webview/browser/gfx/deferred_gpu_command_service.cc:135
#2  android_webview::DeferredGpuCommandService::GetInstance () at ../../android_webview/browser/gfx/deferred_gpu_command_service.cc:148
#3  0xbf03abe6 in android_webview::(anonymous namespace)::GetSyncPointManager () at ../../android_webview/lib/aw_main_delegate.cc:379
#4  0xc130f834 in content::(anonymous namespace)::CreateVizMainDependencies () at ./../../content/gpu/gpu_child_thread.cc:172
#5  content::GpuChildThread::GpuChildThread(base::RepeatingCallback<void ()>, content::ChildThreadImpl::Options const&, std::__1::unique_ptr<gpu::GpuInit, std::__1::default_delete<gpu::GpuInit> >) () at ./../../content/gpu/gpu_child_thread.cc:209
#6  0xc1310abc in content::GpuChildThread::GpuChildThread () at ./../../content/gpu/gpu_child_thread.cc:196
#7  content::InProcessGpuThread::Init () at ./../../content/gpu/in_process_gpu_thread.cc:59
#8  0xbfdb7a52 in base::Thread::ThreadMain () at ./../../base/threading/thread.cc:301
#9  0xbfddb234 in base::(anonymous namespace)::ThreadFunc () at ./../../base/threading/platform_thread_posix.cc:81
```
RenderThread创建AwGLSurface
```
#0  gpu::InProcessCommandBuffer::Initialize () at ../../gpu/ipc/in_process_command_buffer.cc:347
#1  0xbe42f34c in gpu::GLInProcessContext::Initialize () at ../../gpu/ipc/gl_in_process_context.cc:78
#2  0xbcb58f00 in android_webview::AwRenderThreadContextProvider::AwRenderThreadContextProvider ()
    at ../../android_webview/browser/gfx/aw_render_thread_context_provider.cc:69
#3  android_webview::AwRenderThreadContextProvider::Create () at ../../android_webview/browser/gfx/aw_render_thread_context_provider.cc:32
#4  0xbcb5f5f4 in android_webview::SurfacesInstance::SurfacesInstance () at ../../android_webview/browser/gfx/surfaces_instance.cc:129
#5  android_webview::SurfacesInstance::GetOrCreateInstance () at ../../android_webview/browser/gfx/surfaces_instance.cc:55
#6  0xbcb5c3e4 in android_webview::HardwareRenderer::HardwareRenderer () at ../../android_webview/browser/gfx/hardware_renderer.cc:33
#7  0xbcb5df4c in android_webview::RenderThreadManager::DrawOnRT () at ../../android_webview/browser/gfx/render_thread_manager.cc:203
#8  0xbcb5731a in android_webview::AwDrawFnImpl::DrawInternal<AwDrawFn_DrawGLParams> () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:624
#9  android_webview::AwDrawFnImpl::DrawGL () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:275
#10 android_webview::(anonymous namespace)::DrawGLWrapper () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:133
#11 0xde1b6262 in android::(anonymous namespace)::draw_gl(int, void*, android::uirenderer::DrawGlInfo const&) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libwebviewchromium_plat_support.so
#12 0xe79f2e20 in android::uirenderer::WebViewFunctor::drawGl(android::uirenderer::DrawGlInfo const&) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#13 0xe79f256e in android::uirenderer::skiapipeline::GLFunctorDrawable::onDraw(SkCanvas*) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#14 0xe7919024 in SkCanvas::onDrawDrawable(SkDrawable*, SkMatrix const*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#15 0xe79115f4 in android::uirenderer::skiapipeline::RenderNodeDrawable::drawContent(SkCanvas*) const ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#16 0xe7937a42 in android::uirenderer::skiapipeline::RenderNodeDrawable::forceDraw(SkCanvas*) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#17 0xe793ccc8 in android::uirenderer::skiapipeline::SkiaPipeline::renderLayersImpl(android::uirenderer::LayerUpdateQueue const&, bool) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#18 0xe793ae44 in android::uirenderer::skiapipeline::SkiaPipeline::renderFrame(android::uirenderer::LayerUpdateQueue const&, SkRect const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, bool, android::uirenderer::Rect const&, sk_sp<SkSurface>, SkMatrix const&) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#19 0xe793abc0 in android::uirenderer::skiapipeline::SkiaOpenGLPipeline::draw(android::uirenderer::renderthread::Frame const&, SkRect const&, SkRect const&, android::uirenderer::LightGeometry const&, android::uirenderer::LayerUpdateQueue*, android::uirenderer::Rect const&, bool, android::uirenderer::LightInfo const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, android::uirenderer::FrameInfoVisualizer*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#20 0xe7979df6 in android::uirenderer::renderthread::CanvasContext::draw() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#21 0xe79793de in std::__1::__function::__func<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0, std::__1::allocator<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0>, void ()>::operator() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#22 0xe7986cd0 in android::uirenderer::WorkQueue::process() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#23 0xe7986b2a in android::uirenderer::renderthread::RenderThread::threadLoop() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#24 0xe6d2689a in android::Thread::_threadLoop(void*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libutils.so
```
RenderThread 回放DrawQuad
```
#0  gpu::gles2::GLES2Implementation::CreateAndConsumeTextureCHROMIUM () at ./../../gpu/command_buffer/client/gles2_implementation.cc:6433
#1  0xbd3a8ade in viz::DisplayResourceProvider::LockForRead () at ./../../components/viz/service/display/display_resource_provider.cc:478
#2  0xbd3b0d54 in viz::DisplayResourceProvider::ScopedReadLockGL::ScopedReadLockGL () at ./../../components/viz/service/display/display_resource_provider.cc:843
#3  viz::DisplayResourceProvider::ScopedSamplerGL::ScopedSamplerGL () at ./../../components/viz/service/display/display_resource_provider.cc:864
#4  viz::GLRenderer::DrawContentQuadNoAA () at ./../../components/viz/service/display/gl_renderer.cc:2105
#5  viz::GLRenderer::DrawContentQuad () at ./../../components/viz/service/display/gl_renderer.cc:1971
#6  viz::GLRenderer::DrawTileQuad () at ./../../components/viz/service/display/gl_renderer.cc:1938
#7  viz::GLRenderer::DoDrawQuad () at ./../../components/viz/service/display/gl_renderer.cc:533
#8  0xbd3a0886 in viz::DirectRenderer::DrawRenderPass () at ./../../components/viz/service/display/direct_renderer.cc:714
#9  viz::DirectRenderer::DrawRenderPassAndExecuteCopyRequests () at ./../../components/viz/service/display/direct_renderer.cc:573
#10 0xbd3a6ea0 in viz::DirectRenderer::DrawFrame () at ./../../components/viz/service/display/direct_renderer.cc:429
#11 viz::Display::DrawAndSwap () at ./../../components/viz/service/display/display.cc:492
#12 0xbbc04b6e in android_webview::SurfacesInstance::DrawAndSwap () at ../../android_webview/browser/gfx/surfaces_instance.cc:239
#13 0xbbc01e48 in android_webview::HardwareRenderer::Draw () at ../../android_webview/browser/gfx/hardware_renderer.cc:197
#14 0xbbc0301a in android_webview::RenderThreadManager::DrawOnRT () at ../../android_webview/browser/gfx/render_thread_manager.cc:208
#15 0xbbbfc3ca in android_webview::AwDrawFnImpl::DrawInternal<AwDrawFn_DrawGLParams> () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:624
#16 android_webview::AwDrawFnImpl::DrawGL () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:275
#17 android_webview::(anonymous namespace)::DrawGLWrapper () at ../../android_webview/browser/gfx/aw_draw_fn_impl.cc:133
#18 0xdd5e5262 in android::(anonymous namespace)::draw_gl(int, void*, android::uirenderer::DrawGlInfo const&) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libwebviewchromium_plat_support.so
#19 0xe8712e20 in android::uirenderer::WebViewFunctor::drawGl(android::uirenderer::DrawGlInfo const&) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#20 0xe871256e in android::uirenderer::skiapipeline::GLFunctorDrawable::onDraw(SkCanvas*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#21 0xe8639024 in SkCanvas::onDrawDrawable(SkDrawable*, SkMatrix const*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#22 0xe86315f4 in android::uirenderer::skiapipeline::RenderNodeDrawable::drawContent(SkCanvas*) const ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#23 0xe8657a42 in android::uirenderer::skiapipeline::RenderNodeDrawable::forceDraw(SkCanvas*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#24 0xe865ccc8 in android::uirenderer::skiapipeline::SkiaPipeline::renderLayersImpl(android::uirenderer::LayerUpdateQueue const&, bool) ()
   from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#25 0xe865ae44 in android::uirenderer::skiapipeline::SkiaPipeline::renderFrame(android::uirenderer::LayerUpdateQueue const&, SkRect const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, bool, android::uirenderer::Rect const&, sk_sp<SkSurface>, SkMatrix const&) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#26 0xe865abc0 in android::uirenderer::skiapipeline::SkiaOpenGLPipeline::draw(android::uirenderer::renderthread::Frame const&, SkRect const&, SkRect const&, android::uirenderer::LightGeometry const&, android::uirenderer::LayerUpdateQueue*, android::uirenderer::Rect const&, bool, android::uirenderer::LightInfo const&, std::__1::vector<android::sp<android::uirenderer::RenderNode>, std::__1::allocator<android::sp<android::uirenderer::RenderNode> > > const&, android::uirenderer::FrameInfoVisualizer*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#27 0xe8699df6 in android::uirenderer::renderthread::CanvasContext::draw() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#28 0xe86993de in std::__1::__function::__func<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0, std::__1::allocator<android::uirenderer::renderthread::DrawFrameTask::postAndWait()::$_0>, void ()>::operator() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#29 0xe86a6cd0 in android::uirenderer::WorkQueue::process() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#30 0xe86a6b2a in android::uirenderer::renderthread::RenderThread::threadLoop() () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libhwui.so
#31 0xe8ca889a in android::Thread::_threadLoop(void*) () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libutils.so
```
GPU thread 用skia 绘制
```
#0  0xe6692354 in glDrawElements () from /tmp/adb-gdb-libs-99121FFAZ00A7P/system/lib/libGLESv2.so
#1  0xbcefdbcc in GrGLGpu::sendIndexedMeshToGpu () at ../../third_party/skia/include/gpu/gl/GrGLFunctions.h:303
#2  0xbcefdc2e in non-virtual thunk to GrGLGpu::sendIndexedMeshToGpu(GrPrimitiveType, GrBuffer const*, int, int, unsigned short, unsigned short, GrBuffer const*, int, GrPrimitiveRestart) () at ../../third_party/skia/include/gpu/gl/GrGLFunctions.h:303
#3  0xbcefda84 in GrMesh::sendToGpu () at ../../third_party/skia/src/gpu/GrMesh.h:246
#4  0xbcefd92e in GrGLGpu::draw () at ../../third_party/skia/src/gpu/gl/GrGLGpu.cpp:2624
#5  0xbcf015ac in GrGLGpuRTCommandBuffer::onDraw () at ../../third_party/skia/src/gpu/gl/GrGLGpuCommandBuffer.h:88
#6  0xbce88cda in GrGpuRTCommandBuffer::draw () at ../../third_party/skia/src/gpu/GrGpuCommandBuffer.cpp:101
#7  0xbce8a232 in GrOpFlushState::executeDrawsAndUploadsForMeshDrawOp(GrOp const*, SkRect const&, GrProcessorSet&&, GrPipeline::InputFlags, GrUserStencilSettings const*) () at ../../third_party/skia/src/gpu/GrOpFlushState.cpp:56
#8  0xbced1f7a in GrSimpleMeshDrawOpHelper::executeDrawsAndUploads () at ../../third_party/skia/src/gpu/ops/GrSimpleMeshDrawOpHelper.cpp:109
#9  0xbce9c79c in GrOp::execute () at ../../third_party/skia/src/gpu/ops/GrOp.h:181
#10 0xbce9c6a0 in GrRenderTargetOpList::onExecute () at ../../third_party/skia/src/gpu/GrRenderTargetOpList.cpp:512
#11 0xbce8308a in GrOpList::execute () at ../../third_party/skia/include/private/GrOpList.h:41
#12 GrDrawingManager::executeOpLists () at ../../third_party/skia/src/gpu/GrDrawingManager.cpp:496
#13 GrDrawingManager::flush () at ../../third_party/skia/src/gpu/GrDrawingManager.cpp:357
#14 0xbce83464 in GrDrawingManager::flushSurfaces () at ../../third_party/skia/src/gpu/GrDrawingManager.cpp:575
#15 0xbce9b1dc in GrDrawingManager::flushSurface () at ../../third_party/skia/src/gpu/GrDrawingManager.h:89
#16 GrRenderTargetContext::flush () at ../../third_party/skia/src/gpu/GrRenderTargetContext.cpp:1757
#17 0xbbdb209a in SkSurface::flush () at ../../third_party/skia/src/image/SkSurface.cpp:247
---Type <return> to continue, or q <return> to quit---
#18 SkSurface::flush () at ../../third_party/skia/src/image/SkSurface.cpp:243
#19 0xbd4dc312 in gpu::raster::RasterDecoderImpl::DoEndRasterCHROMIUM ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/raster_decoder.cc:2193
#20 gpu::raster::RasterDecoderImpl::HandleEndRasterCHROMIUM () at ../../gpu/command_buffer/service/raster_decoder_autogen.h:157
#21 0xbd4df1a6 in gpu::raster::RasterDecoderImpl::DoCommandsImpl<false> ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/raster_decoder.cc:1184
#22 gpu::raster::RasterDecoderImpl::DoCommands () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/raster_decoder.cc:1243
#23 0xbd3a25a0 in gpu::CommandBufferService::Flush () at ./../../gpu/command_buffer/service/command_buffer_service.cc:69
#24 0xbd53d160 in gpu::CommandBufferStub::OnAsyncFlush () at ./../../gpu/ipc/service/command_buffer_stub.cc:522
```
gpu线程创建skia surface
```
#0  gpu::SharedImageRepresentationSkiaImpl::SharedImageRepresentationSkiaImpl ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:359
#1  0xbd5d91ea in base::RefCounted<gpu::SharedContextState, base::DefaultRefCountedTraits<gpu::SharedContextState> >::DeleteInternal<gpu::SharedContextState> () at ../../base/memory/ref_counted.h:352
#2  base::DefaultRefCountedTraits<gpu::SharedContextState>::Destruct () at ../../base/memory/ref_counted.h:318
#3  base::RefCounted<gpu::SharedContextState, base::DefaultRefCountedTraits<gpu::SharedContextState> >::Release ()
    at ../../base/memory/ref_counted.h:341
#4  scoped_refptr<gpu::SharedContextState>::Release () at ../../base/memory/scoped_refptr.h:297
#5  scoped_refptr<gpu::SharedContextState>::~scoped_refptr () at ../../base/memory/scoped_refptr.h:209
#6  std::__1::make_unique<gpu::SharedImageRepresentationSkiaImpl, gpu::SharedImageManager*&, gpu::SharedImageBackingGLTexture*, scoped_refptr<gpu::SharedContextState>, sk_sp<SkPromiseImageTexture>&, gpu::MemoryTypeTracker*&, unsigned int, unsigned int>(gpu::SharedImageManager*&, gpu::SharedImageBackingGLTexture*&&, scoped_refptr<gpu::SharedContextState>&&, sk_sp<SkPromiseImageTexture>&, gpu::MemoryTypeTracker*&, unsigned int&&, unsigned int&&) () at ../../buildtools/third_party/libc++/trunk/include/memory:3131
#7  gpu::SharedImageBackingGLTexture::ProduceSkia ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:615
#8  0xbd5c925e in base::RefCounted<gpu::SharedContextState, base::DefaultRefCountedTraits<gpu::SharedContextState> >::DeleteInternal<gpu::SharedContextState> () at ../../base/memory/ref_counted.h:352
#9  base::DefaultRefCountedTraits<gpu::SharedContextState>::Destruct () at ../../base/memory/ref_counted.h:318
#10 base::RefCounted<gpu::SharedContextState, base::DefaultRefCountedTraits<gpu::SharedContextState> >::Release ()
    at ../../base/memory/ref_counted.h:341
#11 scoped_refptr<gpu::SharedContextState>::Release () at ../../base/memory/scoped_refptr.h:297
#12 scoped_refptr<gpu::SharedContextState>::~scoped_refptr () at ../../base/memory/scoped_refptr.h:209
#13 gpu::SharedImageManager::ProduceSkia () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_manager.cc:192
#14 0xbd5bb6a2 in base::RefCounted<gpu::SharedContextState, base::DefaultRefCountedTraits<gpu::SharedContextState> >::DeleteInternal<gpu::SharedContextState> () at ../../base/memory/ref_counted.h:352
#15 base::DefaultRefCountedTraits<gpu::SharedContextState>::Destruct () at ../../base/memory/ref_counted.h:318
#16 base::RefCounted<gpu::SharedContextState, base::DefaultRefCountedTraits<gpu::SharedContextState> >::Release ()
    at ../../base/memory/ref_counted.h:341
#17 scoped_refptr<gpu::SharedContextState>::Release () at ../../base/memory/scoped_refptr.h:297
#18 scoped_refptr<gpu::SharedContextState>::~scoped_refptr () at ../../base/memory/scoped_refptr.h:209
#19 gpu::SharedImageRepresentationFactory::ProduceSkia ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:337
#20 gpu::raster::RasterDecoderImpl::DoBeginRasterCHROMIUM ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/raster_decoder.cc:2026
#21 gpu::raster::RasterDecoderImpl::HandleBeginRasterCHROMIUMImmediate () at ../../gpu/command_buffer/service/raster_decoder_autogen.h:130
#22 0xbd5bf1b6 in gpu::raster::RasterDecoderImpl::DoCommandsImpl<false> ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/raster_decoder.cc:1205
#23 gpu::raster::RasterDecoderImpl::DoCommands () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/raster_decoder.cc:1243
#24 0xbd4825b0 in gpu::CommandBufferService::Flush () at ./../../gpu/command_buffer/service/command_buffer_service.cc:75
#25 0xbd61d170 in gpu::CommandBufferStub::OnAsyncFlush () at ./../../gpu/ipc/service/command_buffer_stub.cc:525
#26 0xbd61cace in trace_event_internal::ScopedTracer::ScopedTracer () at ../../base/trace_event/trace_event.h:983
#27 IPC::MessageT<GpuCommandBufferMsg_RegisterTransferBuffer_Meta, std::__1::tuple<int, base::UnsafeSharedMemoryRegion>, void>::Dispatch<gpu::CommandBufferStub, gpu::CommandBufferStub, void, void (gpu::CommandBufferStub::*)(int, base::UnsafeSharedMemoryRegion)> ()
    at ../../ipc/ipc_message_templates.h:143
```
gpu线程创建shared image backing 实际就是GL_TEXTURE_2D
```
#0  gpu::SharedImageBackingFactoryGLTexture::MakeBacking ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:1241
#1  0xbd480cb0 in gpu::SharedImageBackingFactoryGLTexture::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:1052
#2  0xbd480792 in gpu::SharedImageBackingFactoryGLTexture::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:857
#3  0xbd483336 in gpu::SharedImageFactory::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:115
#4  0xbd4df890 in gpu::SharedImageStub::OnCreateSharedImage () at ./../../gpu/ipc/service/shared_image_stub.cc:99
#5  0xbd4df5a0 in base::DispatchToMethodImpl<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>, 0u>(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&, std::__1::integer_sequence<unsigned int, 0u>) ()
    at ../../base/tuple.h:52
```
gpu线程创建 EglImage
```
#0  gpu::gles2::TextureDefinition::TextureDefinition ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/texture_definition.cc:361
#1  0xbd3ad1b0 in gpu::gles2::MailboxManagerSync::ProduceTexture ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/mailbox_manager_sync.cc:250
#2  0xbd3da506 in gpu::SharedImageBackingPassthroughGLTexture::ProduceLegacyMailbox ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_backing_factory_gl_texture.cc:680
#3  0xbd3ca5bc in gpu::SharedImageRepresentationFactoryRef::ProduceLegacyMailbox ()
    at ../../gpu/command_buffer/service/shared_image_representation.h:73
#4  gpu::SharedImageFactory::RegisterBacking () at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:296
#5  0xbd3ca36e in gpu::SharedImageFactory::CreateSharedImage ()
    at ../../third_party/mesa_headers/../../gpu/command_buffer/service/shared_image_factory.cc:120
#6  0xbd426890 in gpu::SharedImageStub::OnCreateSharedImage () at ./../../gpu/ipc/service/shared_image_stub.cc:99
#7  0xbd4265a0 in base::DispatchToMethodImpl<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>, 0u>(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&, std::__1::integer_sequence<unsigned int, 0u>) ()
    at ../../base/tuple.h:52
#8  base::DispatchToMethod<gpu::SharedImageStub*, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params> >(gpu::SharedImageStub* const&, void (gpu::SharedImageStub::*)(GpuChannelMsg_CreateSharedImage_Params const&), std::__1::tuple<GpuChannelMsg_CreateSharedImage_Params>&&) () at ../../base/tuple.h:60
```
eglImage被放入MailboxManagerSync的texture_to_group_里面

然后在RenderThread DrawQuad的时候通过resource id 拿到resource，通过resource的mailbox name,调用CreateAndConsumeTextureCHROMIUM,先序列化，再回放，取到gpu线程绘制的eglImage。 
