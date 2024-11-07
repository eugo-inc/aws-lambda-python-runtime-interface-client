# === @begin: General ===
project(
    'awslambdaric',
    'c', 'cpp',
    version: '3.0.0.dev0',
    default_options: [
        'c_std=gnu17',
        'cpp_std=gnu++23',
        'buildtype=release',
    ],
    meson_version: '>=1.3.0'
)
# === @end: General ===


# === @begin: Meson modules imports ===
fs = import('fs')
py = import('python').find_installation(pure: false)
# === @end: Meson modules imports ===


# === @begin: Dependencies ===
libcurl = dependency('CURL', modules: ['CURL::libcurl'])
aws_lambda_cpp = dependency('aws-lambda-runtime', method: 'cmake', modules: ['AWS::aws-lambda-runtime'])
# === @end: Dependencies ===


# === @begin: Compilable Python modules ===
py.extension_module(
    'runtime_client',

    ['awslambdaric/runtime_client.cpp'],

    install: true,
    subdir: 'awslambdaric/',
    dependencies: [
        # Eugo-managed
        libcurl,
        aws_lambda_cpp,
    ]
)
# === @end: Compilable Python modules ===


# === @begin: Pure Python modules ===
install_subdir(
    'awslambdaric/',
    install_dir: py.get_install_dir(),
    install_tag: 'python-runtime',

    follow_symlinks: true,

    exclude_files: [
        'runtime_client.cpp'
    ]
)
# === @end: Pure Python modules ===
