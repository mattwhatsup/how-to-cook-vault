<%*
/**
 * ============================
 * 菜谱整理脚本（Markdown → @菜谱，图片 → @asset）
 * ============================
 */

const DRY_RUN = true
const SOURCE_FOLDER = "菜谱/"      // 原始菜谱所在路径
const TARGET_MD_FOLDER = "@菜谱/"  // 处理后的 Markdown 目标路径（Vault 根）
const ASSET_FOLDER = "@assets/"     // 图片统一存放
const IMG_EXT = ["png", "jpg", "jpeg", "webp"]

/** 确保目标文件夹存在 */
async function ensureFolder(path) {
  if (!app.vault.getAbstractFileByPath(path)) {
    if (DRY_RUN) {
      console.log(`[DRY RUN] Create folder: ${path}`)
    } else {
    console.log(path)
	    try{
	      await app.vault.createFolder(path)
	    } catch(e) {}
    }
  }
}

await ensureFolder(TARGET_MD_FOLDER)
await ensureFolder(ASSET_FOLDER)

/** 获取所有源菜谱 Markdown（递归） */
const files = app.vault.getFiles().filter(f =>
  f.extension === "md" &&
  f.path.startsWith(SOURCE_FOLDER)
)

console.log(`Found ${files.length} recipe markdown files.`)

/** 提取文档中所有图片引用 */
function extractImages(content) {
  const results = []

  // ![[xxx]]
  const wikiRegex = /!\[\[([^\]]+)\]\]/g
  let m
  while ((m = wikiRegex.exec(content)) !== null) {
    results.push({ raw: m[0], path: m[1] })
  }

  // ![](xxx)
  const mdRegex = /!\[[^\]]*\]\(([^)]+)\)/g
  while ((m = mdRegex.exec(content)) !== null) {
    results.push({ raw: m[0], path: m[1] })
  }

  return results
}

/** 主处理循环 */
for (const file of files) {
console.log(file.path)
  const originalPath = file.path
  const content = await app.vault.read(file)

  const normalized = content.replace(/^\uFEFF/, "").trimStart()
  if (normalized.startsWith("---")) {
    console.log(`Skip (already has YAML): ${originalPath}`)
    continue
  }

  /** 分类 = 菜谱/分类名/xxx.md */
  const parts = originalPath.split("/")
  const category = parts.length >= 3 ? parts[1] : ""

  /** 原 Markdown 所在目录 */
  const baseDir = parts.slice(0, -1).join("/")

  /** 扫描图片 */
  const images = extractImages(content)
  const imageMap = {} // raw -> new raw

  for (const img of images) {
    let imgPath = img.path

    // 跳过网络图片
    if (imgPath.startsWith("http")) continue

    imgPath = imgPath.replace(/^\.\//, "")
    const ext = imgPath.split(".").pop().toLowerCase()
    if (!IMG_EXT.includes(ext)) continue

    const absOldPath = `${baseDir}/${imgPath}`
    const imgFile = app.vault.getAbstractFileByPath(absOldPath)
    if (!imgFile) continue

    let newName = imgFile.name
    let newPath = ASSET_FOLDER + newName

    // 防止重名
    let i = 1
    while (!DRY_RUN && app.vault.getAbstractFileByPath(newPath)) {
      const stem = newName.replace(/\.[^/.]+$/, "")
      newPath = `${ASSET_FOLDER}${stem}-${i}.${ext}`
      i++
    }

    imageMap[img.raw] = `![[${newPath}]]`

    if (DRY_RUN) {
      console.log(`[DRY RUN] Image: ${absOldPath} -> ${newPath}`)
    } else {
    try{
      await app.vault.rename(imgFile, newPath)
      }catch(e) {}
    }
  }

  /** 更新正文图片引用 */
  let updatedContent = content
  for (const [raw, replacement] of Object.entries(imageMap)) {
    updatedContent = updatedContent.split(raw).join(replacement)
  }

  /** snapshot：使用第一张图片（若存在） */
  const firstImg = images.length
    ? images[0].path.split("/").pop()
    : null

  const snapshot = firstImg
    ? `![[${ASSET_FOLDER}${firstImg}]]`
    : null

  /** YAML */
  const YAML_BLOCK = `---
名称: ${file.basename}
分类: ${category}
${snapshot ? `图片: "${snapshot}"` : ""}
---
`

  /** 最终 Markdown 目标路径 */
  const targetMdPath = TARGET_MD_FOLDER + file.name

  if (DRY_RUN) {
    console.log(`[DRY RUN] Markdown: ${originalPath} -> ${targetMdPath}`)
  } else {
    await app.vault.rename(file, targetMdPath)
    const newFile = app.vault.getAbstractFileByPath(targetMdPath)
    await app.vault.modify(newFile, YAML_BLOCK + updatedContent)
  }
}

console.log("Recipe migration to @菜谱 finished.")
%>







