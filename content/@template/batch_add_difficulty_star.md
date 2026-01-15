<%*
/**
 * ============================
 * 根据「难度评星/*.md」为 @菜谱 中的菜谱回填 YAML 难度评星
 * ============================
 */

const DRY_RUN = false

const STAR_FOLDER = "难度评星/"
const RECIPE_FOLDER = "@菜谱/"

/** 工具：读取 YAML 区块 */
function extractYaml(content) {
  const m = content.match(/^---[\s\S]*?^---/m)
  return m ? m[0] : null
}

/** 工具：移除 YAML */
function stripYaml(content) {
  return content.replace(/^---[\s\S]*?^---\s*/m, "")
}

/** 收集：菜谱名 → 难度数字 */


const recipeStarMap = new Map()

const starFiles = app.vault.getFiles().filter(f =>
  f.extension === "md" &&
  f.path.startsWith(STAR_FOLDER) &&
  /^\d+Star\.md$/.test(f.name)
)

console.log(`Found ${starFiles.length} star definition files.`)

/** 解析评星文件 */
for (const starFile of starFiles) {
  const star = parseInt(starFile.name.match(/^(\d+)Star\.md$/)[1], 10)
  const content = await app.vault.read(starFile)

  const lines = content.split("\n")

  for (const line of lines) {
    const m = line.match(/\*\s+\[([^\]]+)\]\(([^)]+)\)/)
    if (!m) continue

    const recipeName = m[1].trim()
    recipeStarMap.set(recipeName, star)
  }
}

console.log(`Collected ${recipeStarMap.size} recipe-star mappings.`)

/** 处理 @菜谱 下的菜谱 */
const recipeFiles = app.vault.getFiles().filter(f =>
  f.extension === "md" &&
  f.path.startsWith(RECIPE_FOLDER)
)

for (const file of recipeFiles) {
  const recipeName = file.basename
  const star = recipeStarMap.get(recipeName)

  if (star === undefined) continue

  const content = await app.vault.read(file)
  const yaml = extractYaml(content)
  const body = stripYaml(content)

  let newYaml = ""

  if (yaml) {
    if (/^难度评星:/m.test(yaml)) {
      newYaml = yaml.replace(
        /^难度评星:\s*.*$/m,
        `难度评星: ${star}`
      )
    } else {
      newYaml = yaml.replace(
        /^---/,
        `---\n难度评星: ${star}`
      )
    }
  } else {
    newYaml = `---\n难度评星: ${star}\n---`
  }

  const newContent = `${newYaml}\n${body}`

  if (DRY_RUN) {
    console.log(
      `[DRY RUN] ${file.path} → 难度评星: ${star}`
    )
  } else {
    await app.vault.modify(file, newContent)
    console.log(
      `[UPDATED] ${file.path} → 难度评星: ${star}`
    )
  }
}

console.log("难度评星回填完成")
%>

