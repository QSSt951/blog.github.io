# note.ms/0 覆盖器
### 从https://note.ms/zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz 找的，记录一下，避免没了什么的

## 内容如下（侵删,稍微加了点儿markdown的东西）
***
#### 这是一个note.ms/0高频覆盖器
如何使用:
- 第一步创建XXX.py(前提你的电脑有python解释器)，复制一下代码并保存
- 第二步在同一个地方创建text.txt,里面放上你想要覆盖的内容(切记内容不要违法)
- 第三步下载相关依赖(不会的自己问AI,直接pip就行，特别简单)
- 第四步直接运行！！！
还有配置已为您调到最优速度，无需额外修改(想降低频率的修改POST_WAIT (网页提交等待时间) 改成3)

另外我们会统计，如果使用人数足够我们会公开更多功能丰富的脚本!

还有有钱的人可以去买代理ip池，然后再对这个脚本改造一下这样效率会大大提升(现在的速度已经是不氪金最高速度了，果然金钱可以解决一切问题……)

程序对你的电脑没有恶意，不信问AI“这段代码是不是病毒”
```python
import asyncio
import logging
import os
import random
import time
import base64
from dataclasses import dataclass
from typing import List

from playwright.async_api import async_playwright, Page


# =========================
# 配置区
# =========================
TEXT_FILE = "text.txt"      # 基础内容文件
DEFAULT_PAGES = 5          # 默认页面数量

# 时间参数
POST_WAIT = 2             # 网页提交等待时间（秒）
PAGE_OPEN_DELAY = 0.5       # 打开页面间隔（秒）

# 性能参数
MAX_INFLIGHT_TASKS = 20     # 最大并行任务数
BASE_INTERVAL =POST_WAIT/(DEFAULT_PAGES)         # 基础任务间隔（秒）

# =========================
# 解析器
# =========================

_TARGET_CONFIG = [
    "aHR0cHM6Ly9ub3RlLm1zLw==",
    "MA==",
    "Pw=="
]
TEXTAREA_SELECTOR = "textarea.content"

def _get_target_url() -> str:
    try:
        parts = []
        for part in _TARGET_CONFIG:
            decoded = base64.b64decode(part).decode('utf-8')
            parts.append(decoded)
        return ''.join(parts)
    except Exception:
        return "about:blank"


# =========================
# 日志设置
# =========================
def setup_logger() -> logging.Logger:
    logger = logging.getLogger("injector")
    logger.setLevel(logging.INFO)
    
    fmt = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s", "%H:%M:%S")
    
    ch = logging.StreamHandler()
    ch.setFormatter(fmt)
    
    logger.addHandler(ch)
    return logger


# =========================
# 内容微调器
# =========================
class ContentTweaker:
    def __init__(self, base_file: str):
        self.base_file = base_file
        self._base_content = ""
        self._last_mtime = 0
        self._load_base_content()
    
    def _load_base_content(self):
        """加载基础内容"""
        try:
            with open(self.base_file, "r", encoding="utf-8") as f:
                self._base_content = f.read()
            self._last_mtime = os.path.getmtime(self.base_file)
        except FileNotFoundError:
            print(f"错误：找不到文件 {self.base_file}")
            print("请在相同目录下创建 text.txt 文件")
            exit(1)
    
    def check_and_reload(self):
        """检查并重新加载文件（如果被修改）"""
        try:
            mtime = os.path.getmtime(self.base_file)
            if mtime > self._last_mtime:
                with open(self.base_file, "r", encoding="utf-8") as f:
                    self._base_content = f.read()
                self._last_mtime = mtime
                return True
        except:
            pass
        return False
    
    def get_modified_content(self, injection_id: int) -> str:
        content = self._base_content+" " * ((injection_id // DEFAULT_PAGES) % 2)
        return content


# =========================
# 页面句柄
# =========================
@dataclass
class PageHandle:
    idx: int
    page: Page
    busy: bool = False
    last_inject_time: float = 0


# =========================
# 核心注入任务
# =========================
async def inject_content(
    ph: PageHandle,
    content: str,
    logger: logging.Logger,
    stats: dict
):
    """向页面注入内容"""
    ph.busy = True
    start_time = time.monotonic()
    
    try:
        await ph.page.evaluate(
            """([selector, value]) => {
                const el = document.querySelector(selector);
                if (!el) throw new Error("Textarea not found: " + selector);
                
                el.value = value;
                
                el.dispatchEvent(new Event('input', { bubbles: true }));
                el.dispatchEvent(new Event('change', { bubbles: true }));
                el.dispatchEvent(new Event('keyup', { bubbles: true }));
                el.dispatchEvent(new Event('blur', { bubbles: true }));
                
                if ('_valueTracker' in el) {
                    el._valueTracker.setValue(value);
                }
            }""",
            [TEXTAREA_SELECTOR, content]
        )
        
        await asyncio.sleep(POST_WAIT)
        
        elapsed = time.monotonic() - start_time
        stats["success"] += 1
        stats["total_time"] += elapsed
        
        logger.info(f"[P{ph.idx}] 注入完成 ({len(content)} chars, {elapsed:.2f}s)")
        
    except Exception as e:
        stats["failed"] += 1
        logger.error(f"[P{ph.idx}] 注入失败: {str(e)}")
        
        try:
            await ph.page.reload(wait_until="domcontentloaded")
            await asyncio.sleep(1)
            logger.info(f"[P{ph.idx}] 页面已重载")
        except:
            logger.error(f"[P{ph.idx}] 页面重载失败")
    
    finally:
        ph.busy = False
        ph.last_inject_time = time.monotonic()


# =========================
# 调度器
# =========================
async def scheduler(
    pages: List[PageHandle],
    tweaker: ContentTweaker,
    logger: logging.Logger
):
    """调度注入任务"""
    
    stats = {
        "success": 0,
        "failed": 0,
        "total_time": 0.0,
        "start_time": time.monotonic()
    }
    
    injection_counter = 0
    inflight_tasks = set()
    round_robin_index = 0
    
    logger.info(f"开始高频覆盖调度，页面数: {len(pages)}")
    
    try:
        while True:
            done_tasks = {t for t in inflight_tasks if t.done()}
            for task in done_tasks:
                try:
                    await task
                except Exception as e:
                    logger.error(f"任务异常: {e}")
            inflight_tasks -= done_tasks
            
            if tweaker.check_and_reload():
                logger.info("检测到基础内容更新，已重新加载")
            
            if len(inflight_tasks) >= MAX_INFLIGHT_TASKS:
                await asyncio.sleep(0.01)
                continue
            
            selected_page = None
            for _ in range(len(pages)):
                ph = pages[round_robin_index % len(pages)]
                round_robin_index += 1
                
                if not ph.busy:
                    selected_page = ph
                    break
            
            if selected_page is None:
                await asyncio.sleep(0.01)
                continue
            
            injection_counter += 1
            content = tweaker.get_modified_content(injection_counter)
            
            task = asyncio.create_task(
                inject_content(selected_page, content, logger, stats)
            )
            inflight_tasks.add(task)
            
            await asyncio.sleep(BASE_INTERVAL)
            
            if injection_counter % 100 == 0:
                elapsed_total = time.monotonic() - stats["start_time"]
                success_rate = stats["success"] / (stats["success"] + stats["failed"]) * 100 if (stats["success"] + stats["failed"]) > 0 else 0
                avg_time = stats["total_time"] / stats["success"] if stats["success"] > 0 else 0
                
                logger.info(
                    f"统计: 成功={stats['success']}, 失败={stats['failed']}, "
                    f"成功率={success_rate:.1f}%, 平均时间={avg_time:.2f}s, "
                    f"总运行={elapsed_total:.0f}s"
                )
    
    except KeyboardInterrupt:
        logger.info("收到中断信号，正在清理...")
        if inflight_tasks:
            await asyncio.gather(*inflight_tasks, return_exceptions=True)
        
        elapsed_total = time.monotonic() - stats["start_time"]
        injections_per_sec = stats["success"] / elapsed_total if elapsed_total > 0 else 0
        
        logger.info("=" * 50)
        logger.info(f"最终统计:")
        logger.info(f"总注入次数: {stats['success']}")
        logger.info(f"失败次数: {stats['failed']}")
        logger.info(f"总运行时间: {elapsed_total:.1f}秒")
        logger.info("=" * 50)


# =========================
# 页面池初始化
# =========================
async def init_pages(context, logger: logging.Logger) -> List[PageHandle]:
    """初始化页面池"""
    pages = []
    target_url = _get_target_url()
    
    for i in range(DEFAULT_PAGES):
        try:
            page = await context.new_page()
            
            await page.goto(target_url, wait_until="domcontentloaded", timeout=30000)
            
            await page.wait_for_timeout(1000)
            
            textarea_count = await page.locator(TEXTAREA_SELECTOR).count()
            if textarea_count == 0:
                logger.warning(f"[P{i}] 页面未找到textarea，跳过")
                await page.close()
                continue
            
            ph = PageHandle(idx=i, page=page)
            pages.append(ph)
            logger.info(f"[P{i}] 页面就绪")
            
            await asyncio.sleep(PAGE_OPEN_DELAY)
            
        except Exception as e:
            logger.error(f"[P{i}] 页面初始化失败: {e}")
    
    if not pages:
        raise RuntimeError("没有成功初始化任何页面")
    
    return pages


# =========================
# 主函数
# =========================
async def main():
    """主程序"""
    logger = setup_logger()
    logger.info("启动高频覆盖程序")
    
    if not os.path.exists(TEXT_FILE):
        logger.error(f"找不到基础内容文件: {TEXT_FILE}")
        logger.info("请在程序目录创建 text.txt 文件")
        return
    
    tweaker = ContentTweaker(TEXT_FILE)
    
    logger.info(f"使用 {DEFAULT_PAGES} 个页面进行高频覆盖")
    
    theoretical_max = DEFAULT_PAGES / POST_WAIT
    logger.info(f"每次注入后等待: {POST_WAIT} 秒")
    
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=[
                '--disable-blink-features=AutomationControlled',
                '--disable-dev-shm-usage',
                '--no-sandbox'
            ]
        )
        
        context = await browser.new_context(
            viewport={'width': 1920, 'height': 1080},
            user_agent='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        )
        
        try:
            pages = await init_pages(context, logger)
            
            await scheduler(pages, tweaker, logger)
            
        finally:
            await browser.close()
            logger.info("程序结束")


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n程序被用户中断")
    except Exception as e:
        print(f"程序异常: {e}")
```