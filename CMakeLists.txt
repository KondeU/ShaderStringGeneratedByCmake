cmake_minimum_required(VERSION 3.10)
project(project_configure_file)

macro(generate_shader shader_file shader_content)
file(READ ${shader_file} ${shader_content})
endmacro()

generate_shader(shader.vs SHADER_VS)
generate_shader(shader.fs SHADER_FS)

configure_file(Shaders.h Shaders.generated.h)
