---
title: Linux GPU系列-04-ARM MALI GPU OpenGL端到端流程
author: Joy Xu
date: 2021-05-12 07:08:16
tags: [linux, gpu]
---

前面已经讲过了整个软件堆栈，MALI GPU工作流程，这次用一个简单的OpenGL的例子讲讲端到端的整体流程。

例子的[完整源码](https://github.com/joyxu/opengl-misc/blob/master/OpenGL/triangle/triangle.cpp)
下面是关键片段的代码：

		float vertices[] = {
			-0.5f, -0.5f, 0.0f, // left
			0.5f, -0.5f, 0.0f, // right
			0.0f,  0.5f, 0.0f  // top
		};

		unsigned int VBO, VAO;
		glGenVertexArrays(1, &VAO);
		glGenBuffers(1, &VBO);
		// bind the Vertex Array Object first, then bind and set vertex buffer(s), and then configure vertex attributes(s).
		glBindVertexArray(VAO);

		glBindBuffer(GL_ARRAY_BUFFER, VBO);
		glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

		glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
		glEnableVertexAttribArray(1);

		glBindBuffer(GL_ARRAY_BUFFER, 0);
		glBindVertexArray(0);

		// render loop
		// -----------
		while (!glfwWindowShouldClose(window))
		{
			// render
			// ------
			glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
			glClear(GL_COLOR_BUFFER_BIT);

			// draw our first triangle
			glUseProgram(shaderProgram);
			glBindVertexArray(VAO); // seeing as we only have a single VAO there's no need to bind it every time, but we'll do so to keep things a bit more organized
			glDrawArrays(GL_TRIANGLES, 0, 3);

			// glfw: swap buffers and poll IO events (keys pressed/released, mouse moved etc.)
			// -------------------------------------------------------------------------------
			glfwSwapBuffers(window);
			glfwPollEvents();
		}

## 准备Vertext数据

顶点数据通过glBufferData拷贝给VBO，VBO之前已经通过glBindBuffer/glBindVertextArray和VAO绑定，这些数据通过
以下流程拷贝到host内存，准备给GPU使用，这部分代码都在MESA中：

	glBufferData
	  main::_mesa_BufferData
	    gl_context::Driver.BufferData
	    	state_trackder::st_bufferobj_data
		  state_tracker::bufferobj_data
		    util::pipe_buffer_write
		      pipe_context::buffer_subdata
		        panfrost:u_default_buffer_subdata //各GPU厂商实现，panfrost使用MESA公共函数


## 准备job

顶点数据放到buffer之后，就通过glDrawArrays绘制一个三角形，最终组织成一个job，流程如下，这部分代码仍然在MESA中：

	glDrawArrays
	  main::_mesa_DrawArrays
	    gl_context::Driver.DrawGallium
	      state_tracker::st_draw_gallium
	        cso_context:cso_multi_draw
		  u_vbuf::u_vbuf_draw_vbo / pipe_context:draw_vbo
		    panfrost::panfrost_draw_emit_vertext
		      panforst::panfrost_emit_draw_descs
		    panfrost::panfrost_emit_vertext_tiler_jobs
		      panfrost__panfrost_add_job


## 提交job

再通过glfwSwapBuffers(glFlush)把之前准备好的job提交给GPU硬件，这里涉及到和DRM、Kernel驱动的交互，具体流程如下：

	glfwSwapBuffers
	  main::_mesa_Flush
	    gl_context::Driver.Flush
	      state_tracker::st_glFlush
	        draw_do_flush
		  panfrost::panfrost_flush
		    panfrost::panfrost_batch_submit_jobs
		      panfrost::panfrost_batch_submit_ioctl
		        drm::drmIoctl(DRM_IOCTL_PANFROST_SUBMIT)

陷入到Kernel之后，由drivers/gpu/drm/panfrost驱动接管，把之前的准备好的job提交给GPU：

	panfrost_ioctl_submit
	  panfrost_job_push
	    drm_sched_entity_push_job
	      drm_sched_rq_add_entity
	        panfrost_job_run
		  panfrost_job_hw_submit
		    panfrost_job_write_affinity （设置job和core的亲和性）
		      job_write （写job manager寄存器）
