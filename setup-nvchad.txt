#!/bin/bash

# BW="$(pwd)"
URL="https://github.com/NvChad/NvChad"
CONFIG="$HOME/.config/nvim"
SHARE="$HOME/.local/share/nvim"

if ! find "$CONFIG" &>/dev/null; then
  git clone "$URL" "$CONFIG" --depth 1 && nvim
else
  pushd "$CONFIG" || exit
  if [[ "$(git config --get remote.origin.url)" == "${URL}" ]]; then
    git pull
    popd || exit
  else
    popd || exit
    echo "create backup"
    mv -v "$CONFIG" "${CONFIG}_$(date +%s)"
    rm "$SHARE" -rf
    git clone "$URL" "$CONFIG" --depth 1 && nvim
  fi
fi

pushd "$CONFIG/lua" || exit
mkdir -pv "custom/plugins"
pushd "custom" || exit

echo "initialize custom"
cat <<EOF > init.lua
local autocmd = vim.api.nvim_create_autocmd

-- Auto resize panes when resizing nvim window
autocmd("VimResized", {
  pattern = "*",
  command = "tabdo wincmd =",
})
EOF

echo "create custom chadrc"
cat <<EOF > chadrc.lua
local M = {}

M.ui = {
  theme_toggle = { "onedark", "one_light" },
  theme = "onedark",
}

M.plugins = require "custom.plugins"

-- check core.mappings for table structure
M.mappings = require "custom.mappings"

return M
EOF

echo "create file for custom mappings"
cat <<EOF > mappings.lua
local M = {}

M.general = {
  n = {
    [";"] = { ":", "command mode", opts = { nowait = true } },
  },
}

-- more keybinds!

return M
EOF

pushd plugins || exit

echo "initialize custom plugins"
cat <<EOF > init.lua
local overrides = require "custom.plugins.overrides"

return {
  ["goolord/alpha-nvim"] = { disable = false }, -- enables dashboard
  -- Override plugin definition options
  ["neovim/nvim-lspconfig"] = {
    config = function()
      require "plugins.configs.lspconfig"
      require "custom.plugins.lspconfig"
    end,
  },

  -- override plugin configs
  ["nvim-treesitter/nvim-treesitter"] = {
    override_options = overrides.treesitter,
  },
  ["williamboman/mason.nvim"] = {
    override_options = overrides.mason,
  },

  -- Install a plugin
  ["max397574/better-escape.nvim"] = {
    event = "InsertEnter",
    config = function()
      require("better_escape").setup()
    end,
  },

  -- code formatting, linting etc
  ["jose-elias-alvarez/null-ls.nvim"] = {
    after = "nvim-lspconfig",
    config = function()
      require "custom.plugins.null-ls"
    end,
  },

  ["Pocco81/auto-save.nvim"] = {
	  config = function()
		  require("auto-save").setup {
			  -- your config goes here
			  -- or just leave it empty :)
		  }
	  end,
  },

  -- remove plugin
  -- ["hrsh7th/cmp-path"] = false,
}

EOF

echo "create custom for for lspconfig"
cat <<EOF > lspconfig.lua
local on_attach = require("plugins.configs.lspconfig").on_attach
local capabilities = require("plugins.configs.lspconfig").capabilities

local lspconfig = require "lspconfig"

local servers = {"clangd", "bashls"} -- type of lspconfig for full list in https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md

capabilities.textDocument.completion.completionItem.snippetSupport = true

for _, lsp in ipairs(servers) do
  lspconfig[lsp].setup {
    on_attach = on_attach,
    capabilities = capabilities,
  }
end

EOF

echo "create default null-ls config"

cat <<EOF > null-ls.lua
local present, null_ls = pcall(require, "null-ls")

if not present then
  return
end

local b = null_ls.builtins

local sources = {

  -- webdev stuff
--  b.formatting.deno_fmt,
  b.formatting.prettier.with { filetypes = { "html", "markdown", "css" } },

  -- Lua
  b.formatting.stylua,

  -- Shell
  b.formatting.shfmt,
  b.diagnostics.shellcheck.with { diagnostics_format = "#{m} [#{c}]" },

  -- cpp
  --b.formatting.clang_format,
  b.formatting.rustfmt,
}

null_ls.setup {
  debug = true,
  sources = sources,
}

EOF

echo "create overrides config"
cat <<EOF > overrides.lua
local M = {}

M.treesitter = {
  ensure_installed = {
    "vim",
    "lua",
    "html",
    "css",
    "typescript",
    "c",
  },
}

M.mason = {
  ensure_installed = {
    -- lua stuff
    "lua-language-server",
    "stylua",

    -- web dev stuff
    --"css-lsp",
    --"html-lsp",
    --"typescript-language-server",
    --"deno",
  },
}

-- git support in nvimtree
M.nvimtree = {
  git = {
    enable = true,
  },

  renderer = {
    highlight_git = true,
    icons = {
      show = {
        git = true,
      },
    },
  },
}

return M

EOF

popd && popd && popd || exit

nvim +PackerSync
