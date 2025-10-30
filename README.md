```
# 1. 【ローカル設定の削除】
# ※変更した可能性のあるリポジトリのフォルダ内で実行してください
git config --local --unset core.autocrlf

# 2. 【グローバル設定の削除】
git config --global --unset core.autocrlf


# --- 以下は確認用のコマンドです ---

# 3. 【最終的な設定の確認】
# --system の設定値（おそらく "true"）が表示されればOKです
echo "--- 現在適用されている設定: ---"
git config --get core.autocrlf

# 4. 【（参考）各レベルの確認】
# --system 以外は何も表示されないはずです
echo "--- system (デフォルト): ---"
git config --system --get core.autocrlf

echo "--- global (削除済み): ---"
git config --global --get core.autocrlf

echo "--- local (削除済み): ---"
git config --local --get core.autocrlf
```
