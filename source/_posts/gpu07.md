---
title: Linux GPU系列-07-MESA Gallium MALI Panfrost GPU驱动
author: Joy Xu
date: 2021-05-13 11:57:09
tags: [linux, gpu]
---

# MALI Panfrost 驱动

代码路径在: src/gallium/drivers/panfrost下，官网目前还不太支持Vulkan

最新的开发活动都在：https://gitlab.freedesktop.org/panfrost/mesa
里面有部分Vulkan的支持。

主要还是实现pipe screen和pipe context，screen主要体现硬件的能力，创建和管理资源；
context可以认为是硬件的一个pipe line的实例，涉及到state的设置，fence等。
另外让pipe loader选择我们新创建的pipe driver。

![pipe screen/context/resource用法](/images/driver_pipeline.png)

## 文件说明

主要有两个文件夹：

`src/gallium/drivers/panfrost`：

* pan_screen: 实现了pipe_screen
* pan_context: 实现了pipe_context
* pan_resource: 资源相关的放在这里
* pan_job: 和硬件相关的job放在这里

`src/panfrost/lib`, 由于当前Vulkan还没有公共框架，所以和Vulkan公共的代码放在这个文件夹下：
* pan_device: 设备管理
* pan_bo: buffer object管理
* pan_shader: shader管理
* pan_tiler:tile管理
* pan_texture: texture管理

## pan_screen

定义很简单，主要继承下Gallium和device

		struct panfrost_screen {
			struct pipe_screen base;
			struct panfrost_device dev;
		};


## pan_context

关键实现如下：

		struct panfrost_context {
			/* Gallium context */
			struct pipe_context base;

			/* Upload manager for small resident GPU-internal data structures, like
			 * sampler descriptors. We use an upload manager since the minimum BO
			 * size from the kernel is 4kb */
			struct u_upload_mgr *state_uploader;

			/* Sync obj used to keep track of in-flight jobs. */
			uint32_t syncobj;

			/* Bound job batch and map of panfrost_batch_key to job batches */
			struct panfrost_batch *batch;
			struct hash_table *batches;

			/* panfrost_bo -> panfrost_bo_access */
			struct hash_table *accessed_bos;
			struct pipe_framebuffer_state pipe_framebuffer;
			struct panfrost_streamout streamout;
			struct panfrost_constant_buffer constant_buffer[PIPE_SHADER_TYPES];
			struct panfrost_rasterizer *rasterizer;
			struct panfrost_shader_variants *shader[PIPE_SHADER_TYPES];
			struct panfrost_vertex_state *vertex;
			struct pipe_vertex_buffer vertex_buffers[PIPE_MAX_ATTRIBS];
			struct pipe_shader_buffer ssbo[PIPE_SHADER_TYPES][PIPE_MAX_SHADER_BUFFERS];
			struct pipe_image_view images[PIPE_SHADER_TYPES][PIPE_MAX_SHADER_IMAGES];
			struct panfrost_sampler_state *samplers[PIPE_SHADER_TYPES][PIPE_MAX_SAMPLERS];
			struct panfrost_sampler_view *sampler_views[PIPE_SHADER_TYPES][PIPE_MAX_SHADER_SAMPLER_VIEWS];
			struct primconvert_context *primconvert;
			struct blitter_context *blitter;
			struct panfrost_blend_state *blend;
			struct pipe_viewport_state pipe_viewport;
			struct pipe_scissor_state scissor;
			struct pipe_blend_color blend_color;
			struct panfrost_zsa_state *depth_stencil;
			struct pipe_stencil_ref stencil_ref;
			...
		};

基本严格按照渲染的pipeline来的，直观一点如下图：

![pipe_contex](/images/pipe_context.png)

## pan_resource

引用下pipe_resource，通过pipe_screen的pipe_resource和pan_screen关联起来

		struct panfrost_resource {
			struct pipe_resource base;
			...
		};

		void panfrost_resource_screen_init(struct pipe_screen *pscreen)
		{
			struct panfrost_device *dev = pan_device(pscreen);

			bool fake_rgtc = !panfrost_supports_compressed_format(dev, MALI_BC4_UNORM);

			pscreen->resource_create_with_modifiers =
				panfrost_resource_create_with_modifiers;
			pscreen->resource_create = u_transfer_helper_resource_create;
			pscreen->resource_destroy = u_transfer_helper_resource_destroy;
			pscreen->resource_from_handle = panfrost_resource_from_handle;
			pscreen->resource_get_handle = panfrost_resource_get_handle;
			pscreen->transfer_helper = u_transfer_helper_create(&transfer_vtbl,
					true, false,
					fake_rgtc, true);
		}


pipe_context类似：

		void panfrost_resource_context_init(struct pipe_context *pctx)
		{
			pctx->transfer_map = u_transfer_helper_transfer_map;
			pctx->transfer_unmap = u_transfer_helper_transfer_unmap;
			pctx->create_surface = panfrost_create_surface;
			pctx->surface_destroy = panfrost_surface_destroy;
			pctx->resource_copy_region = util_resource_copy_region;
			pctx->blit = panfrost_blit;
			pctx->generate_mipmap = panfrost_generate_mipmap;
			pctx->flush_resource = panfrost_flush_resource;
			pctx->invalidate_resource = panfrost_invalidate_resource;
			pctx->transfer_flush_region = u_transfer_helper_transfer_flush_region;
			pctx->buffer_subdata = u_default_buffer_subdata;
			pctx->texture_subdata = u_default_texture_subdata;
		}


## pan_job

job都是一批一批提交的，所以主要是panfrost_batch结构，一般属于某个context，实现和硬件强相关。

		struct panfrost_batch {
			struct panfrost_context *ctx;
			...
		}

