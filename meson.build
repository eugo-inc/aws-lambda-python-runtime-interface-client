# === @begin: General ===
project(
    'awslambdaric',
    'c', 'cpp',
    default_options: [
        'c_std=gnu17',
        'cpp_std=gnu++23',
        'buildtype=release',
    ],
    meson_version: '>=1.3.0',
    version: run_command([
        'sed', '-n',
        's/__version__\\s*=\\s*[\'"]\\([^\'\"]*\\)[\'\"]/\\1/p',
        meson.project_source_root() / 'awslambdaric' / '__init__.py'
    ], check: true).stdout().strip()
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

    dependencies: [
        # Eugo-managed
        libcurl,
        aws_lambda_cpp,
    ],
    install: true,
    subdir: 'awslambdaric/' # Originally, they installed that into site_packages root, that is really prone to collisions with other packages as `runtime_client` is quite common noun.
)
# === @end: Compilable Python modules ===


# === @begin: Pure Python modules ===
install_subdir(
    'awslambdaric/',
    install_dir: py.get_install_dir() / 'awslambdaric',
    install_tag: 'python-runtime',

    follow_symlinks: true,

    exclude_files: [
        'runtime_client.cpp'
    ],
)
# === @end: Pure Python modules ===
