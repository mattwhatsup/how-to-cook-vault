
<%*
/**
 * 修复 @菜谱 下 YAML 中的 `图片` 属性
 * ![[xxx]] → xxx
 */

const DRY_RUN = true


const TARGET_FOLDER = "@菜谱/"

const files = app.vault.getFiles().filter(f =>
  f.extension === "md" &&
  f.path.startsWith(TARGET_FOLDER)
)

console.log(`Found ${files.length} recipe files.`)

for (const file of files) {
  const content = await app.vault.read(file)
  // 只处理有 YAML 的文件
  if (!content.startsWith("---")) continue

  const match = content.match(
    /^---[\s\S]*?^---/m
  )
  if (!match) continue

  const yaml = match[0]
  let updatedYaml = yaml

  /**
   * 匹配：
   * 图片: ![[xxx]]
   */
  updatedYaml = updatedYaml.replace(
    /^图片:\s*["']?!?\[\[([^\]]+)\]\]["']?/m,
    (full, path) => {
      console.log(
        `[FIX] ${file.path}: ${full} → 图片: ${path}`
      )
      return `图片: "${path}"`
    }
  )


  if (updatedYaml === yaml) continue

  const updatedContent =
    updatedYaml + content.slice(yaml.length)

  if (DRY_RUN) {
    console.log(`[DRY RUN] Would update: ${file.path}`)
  } else {
    await app.vault.modify(file, updatedContent)
  }
}

console.log("图片属性修复完成")
%>


