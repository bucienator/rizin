project('blake3', 'c', version: '1.3.1', license : ['CC0-1.0'], meson_version: '>=0.55.0')

cc = meson.get_compiler('c')

# handle blake3 dependency
blake3_files = [
  'c' / 'blake3_portable.c',
  'c' / 'blake3_dispatch.c',
  'c' / 'blake3.c',
]

blake3_simd = []

simd_avx2 = '''
#if defined(_MSC_VER)
#include <intrin.h>
#endif
#include <immintrin.h>
int main (int argc, char *argv[]) { __m256i add0 = _mm256_set_epi32(7, 6, 5, 4, 3, 2, 1, 0); return 0; }
'''

simd_avx512 = '''
#if defined(_MSC_VER)
#include <intrin.h>
#endif
#include <immintrin.h>
int main (int argc, char *argv[]) { __m256i add0 = _mm256_setr_epi64x(0, 1, 2, 3); return 0; }
'''

simd_sse2 = '''
#if defined(_MSC_VER)
#include <intrin.h>
#endif
#include <immintrin.h>
int main (int argc, char *argv[]) { __m128i add0 = _mm_set_epi32(3, 2, 1, 0); return 0; }
'''

simd_sse41 = '''
#if defined(_MSC_VER)
#include <intrin.h>
#endif
#include <immintrin.h>
int main (int argc, char *argv[]) { __m128i add0 = _mm_set_epi32(3, 2, 1, 0); return 0; }
'''

simd_neon = '''
#include <immintrin.h>
#include <arm_neon.h>
uint32x4_t set1_128(uint32_t x) { return vld1q_dup_u32(&x); }
int main (int argc, char *argv[]) { uint32x4_t three = set1_128(3); return 0; }
'''

# msvc can't just compile assembly so we use the intrins directly
# https://github.com/mesonbuild/meson/issues/9889
win_msvc_asm = [
  ['c' / 'blake3_avx2.c',   '-DBLAKE3_NO_AVX2',   'avx2',   simd_avx2,   ['/arch:AVX2']],
  ['c' / 'blake3_avx512.c', '-DBLAKE3_NO_AVX512', 'avx512', simd_avx512, ['/arch:AVX512']],
  ['c' / 'blake3_sse2.c',   '-DBLAKE3_NO_SSE2',   'sse2',   simd_sse2,   []], # sse2 extensions are enabled by default
  ['c' / 'blake3_sse41.c',  '-DBLAKE3_NO_SSE41',  'sse41',  simd_sse41,  []], # sse41 extensions are enabled by default
]

win_gnu_asm = [
  ['c' / 'blake3_avx2_x86-64_windows_gnu.S',   '-DBLAKE3_NO_AVX2',   'avx2',   simd_avx2,   ['-mavx2']],
  ['c' / 'blake3_avx512_x86-64_windows_gnu.S', '-DBLAKE3_NO_AVX512', 'avx512', simd_avx512, ['-mavx512f', '-mavx512vl']],
  ['c' / 'blake3_sse2_x86-64_windows_gnu.S',   '-DBLAKE3_NO_SSE2',   'sse2',   simd_sse2,   ['-msse2']],
  ['c' / 'blake3_sse41_x86-64_windows_gnu.S',  '-DBLAKE3_NO_SSE41',  'sse41',  simd_sse41,  ['-msse4.1']],
]

unix_asm = [
  ['c' / 'blake3_avx2_x86-64_unix.S',   '-DBLAKE3_NO_AVX2',   'avx2',   simd_avx2,   ['-mavx2']],
  ['c' / 'blake3_avx512_x86-64_unix.S', '-DBLAKE3_NO_AVX512', 'avx512', simd_avx512, ['-mavx512f', '-mavx512vl']],
  ['c' / 'blake3_sse2_x86-64_unix.S',   '-DBLAKE3_NO_SSE2',   'sse2',   simd_sse2,   ['-msse2']],
  ['c' / 'blake3_sse41_x86-64_unix.S',  '-DBLAKE3_NO_SSE41',  'sse41',  simd_sse41,  ['-msse4.1']],
]

is_x64 = target_machine.cpu_family() == 'x86_64'
is_arm = target_machine.cpu_family() == 'aarch64'
has_intrins = cc.has_header('immintrin.h')

if has_intrins and is_arm
  has_neon = cc.compiles(simd_neon, args: [], name: 'neon')
  if has_neon
    blake3_files += 'c' / 'blake3_neon.c'
    add_project_arguments('-DBLAKE3_USE_NEON=1', language: 'c')
  else
    add_project_arguments('-DBLAKE3_USE_NEON=0', language: 'c')
  endif
elif has_intrins and is_x64
  if target_machine.system() == 'windows'
    if cc.get_id() == 'msvc'
      blake3_simd = win_msvc_asm
    else # gcc or clang
      blake3_simd = win_gnu_asm
    endif
  else # unix only
    blake3_simd = unix_asm
  endif
  add_project_arguments('-DBLAKE3_USE_NEON=0', language: 'c')
else
  # since we do not support SIMD we just use the portable code.
  blake3_flags = [
    '-DBLAKE3_NO_SSE2',
    '-DBLAKE3_NO_SSE41',
    '-DBLAKE3_NO_AVX2',
    '-DBLAKE3_NO_AVX512',
    '-DBLAKE3_USE_NEON=0',
  ]
  foreach flag : blake3_flags
    add_project_arguments(flag, language: 'c')
  endforeach
endif

foreach it : blake3_simd
  file = it[0]
  flag = it[1]
  name = it[2]
  code = it[3]
  args = it[4]
  has_extension = cc.compiles(code, args: args, name: name)
  if has_extension
    blake3_files += file
  else
    add_project_arguments(flag, language: 'c')
  endif
endforeach

blake3_inc = [
  include_directories(['c'])
]

blake3 = static_library('blake3', blake3_files,
  dependencies: [],
  include_directories: blake3_inc,
  implicit_include_directories: false
)

blake3_dep = declare_dependency(
  link_with: blake3,
  include_directories: blake3_inc
)
