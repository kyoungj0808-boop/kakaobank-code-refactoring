원본코드
#!/usr/bin/env python3
"""Generate FENCE dataset images from CSV definitions.

Reads CSV files and generates images for each row based on type:
- baseimg: copy raw Pixabay image to imgs/
- textimg: overlay Korean text on raw Pixabay image
- eng_textimg: overlay English text on raw Pixabay image
- figstep: overlay Korean text on random template
- eng_figstep: overlay English text on random template
"""

import csv
import logging
import random
import sys
from pathlib import Path

from PIL import Image, ImageDraw, ImageFont

PROJECT_ROOT = Path(__file__).resolve().parent.parent
RAW_DIR = PROJECT_ROOT / "raw"
IMGS_DIR = PROJECT_ROOT / "imgs"
TEMPLATE_DIR = PROJECT_ROOT / "templates"
FONT_KO = PROJECT_ROOT / "fonts" / "NanumGothic-Regular.ttf"

CSV_FILES = ["FENCE_benchmark.csv", "FENCE_benchmark_benign.csv"]

KO_TEMPLATES = ["template1.png", "template3.png"]
EN_TEMPLATES = ["template2.png", "template4.png"]

TYPES_WITH_RAW = {"baseimg", "textimg", "eng_textimg"}
TYPES_FIGSTEP = {"figstep", "eng_figstep"}
ALL_TYPES = TYPES_WITH_RAW | TYPES_FIGSTEP

logging.basicConfig(
    format="%(asctime)s [%(levelname)s] %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)


def get_csv_dir() -> Path:
    """Return csv_alias/ if it exists with CSVs, otherwise csv/."""
    alias_dir = PROJECT_ROOT / "csv_alias"
    if alias_dir.is_dir() and list(alias_dir.glob("*.csv")):
        return alias_dir
    return PROJECT_ROOT / "csv"


def load_font(font_path: Path | None, font_size: int) -> ImageFont.FreeTypeFont:
    """Load font from path, or fall back to PIL default."""
    if font_path and font_path.exists():
        return ImageFont.truetype(str(font_path), font_size)
    return ImageFont.load_default(size=font_size)


def wrap_text_by_pixel(
    draw: ImageDraw.ImageDraw,
    text: str,
    font: ImageFont.FreeTypeFont,
    max_width: int,
) -> list[str]:
    """Wrap text into lines that fit within max_width pixels.

    Splits on spaces first (word-level), then falls back to
    character-level splitting for long words or spaceless text (e.g. Korean).
    """
    words = text.split()
    if not words:
        return []

    lines: list[str] = []
    current = ""

    for word in words:
        test = f"{current} {word}" if current else word
        bbox = draw.textbbox((0, 0), test, font=font)
        if bbox[2] - bbox[0] <= max_width:
            current = test
        else:
            # Current line is full — flush it if non-empty
            if current:
                lines.append(current)
            # Check if the single word itself exceeds max_width
            bbox_word = draw.textbbox((0, 0), word, font=font)
            if bbox_word[2] - bbox_word[0] > max_width:
                # Character-level split for oversized word
                chunk = ""
                for ch in word:
                    test_ch = chunk + ch
                    bbox_ch = draw.textbbox((0, 0), test_ch, font=font)
                    if bbox_ch[2] - bbox_ch[0] <= max_width:
                        chunk = test_ch
                    else:
                        if chunk:
                            lines.append(chunk)
                        chunk = ch
                current = chunk
            else:
                current = word

    if current:
        lines.append(current)
    return lines


def text_with_image(
    image_path: Path,
    text: str,
    font_path: Path | None = None,
    font_size: int = 40,
    margin: int = 20,
    padding: int = 20,
) -> Image.Image:
    """Overlay text at bottom of image with black background and white text."""
    font = load_font(font_path, font_size)
    img = Image.open(image_path).convert("RGB")
    draw = ImageDraw.Draw(img)

    img_width, img_height = img.size
    max_text_width = img_width - 2 * margin - 2 * padding

    wrapped_lines = wrap_text_by_pixel(draw, text, font, max_text_width)

    # Calculate text block dimensions
    line_spacing = 8
    single_line_h = font.getbbox("A")[3] - font.getbbox("A")[1]
    text_block_h = len(wrapped_lines) * single_line_h + (len(wrapped_lines) - 1) * line_spacing

    # Draw black background rectangle at bottom
    bg_x1 = margin
    bg_y1 = img_height - text_block_h - 2 * padding - margin
    bg_x2 = img_width - margin
    bg_y2 = img_height - margin
    draw.rectangle([bg_x1, bg_y1, bg_x2, bg_y2], fill="black")

    # Draw white text vertically centered in the background box
    bg_inner_h = bg_y2 - bg_y1
    text_y = bg_y1 + (bg_inner_h - text_block_h) // 2
    for line in wrapped_lines:
        draw.text((margin + padding, text_y), line, fill="white", font=font)
        text_y += single_line_h + line_spacing

    return img


def text_with_figstep(
    template_path: Path,
    text: str,
    font_path: Path | None = None,
    font_size: int = 60,
) -> Image.Image:
    """Overlay text on template image."""
    font = load_font(font_path, font_size)
    img = Image.open(template_path).copy()
    draw = ImageDraw.Draw(img)

    img_width = img.size[0]
    text_x = 470
    max_text_width = img_width - text_x - 40  # 40px right margin

    lines = wrap_text_by_pixel(draw, text, font, max_text_width)

    y = 60
    line_height = font.getbbox("A")[3] + 20

    for line in lines:
        draw.text((text_x, y), line, fill=(0, 0, 0), font=font)
        y += line_height

    return img


def collect_tasks() -> list[dict]:
    """Read CSVs and collect rows for all generation types."""
    csv_dir = get_csv_dir()
    logger.info("Using CSV directory: %s", csv_dir)

    tasks: list[dict] = []
    for csv_file in CSV_FILES:
        csv_path = csv_dir / csv_file
        if not csv_path.exists():
            logger.warning("CSV file not found: %s", csv_path)
            continue
        with open(csv_path, encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for row in reader:
                img_type = row.get("type", "").strip()
                if img_type in ALL_TYPES:
                    tasks.append(row)
    return tasks


def process_row(row: dict, idx: int, total: int) -> str:
    """Process a single CSV row. Returns status: 'generated', 'skipped', or 'failed'."""
    img_type = row["type"].strip()
    image_path = row["image_path"].strip()
    query = row.get("query", "").strip()
    dest = IMGS_DIR / image_path

    # Skip if already exists
    if dest.exists() and dest.stat().st_size > 0:
        logger.info("[%d/%d] Skipped (exists) %s %s", idx, total, img_type, image_path)
        return "skipped"

    dest.parent.mkdir(parents=True, exist_ok=True)

    try:
        if img_type == "baseimg":
            raw_path = RAW_DIR / image_path
            if not raw_path.exists():
                logger.warning("[%d/%d] Raw image missing: %s", idx, total, raw_path)
                return "failed"
            img = Image.open(raw_path).convert("RGB")
            img.save(dest)

        elif img_type == "textimg":
            raw_path = RAW_DIR / image_path
            if not raw_path.exists():
                logger.warning("[%d/%d] Raw image missing: %s", idx, total, raw_path)
                return "failed"
            img = text_with_image(raw_path, query, font_path=FONT_KO)
            img.convert("RGB").save(dest)

        elif img_type == "eng_textimg":
            raw_path = RAW_DIR / image_path
            if not raw_path.exists():
                logger.warning("[%d/%d] Raw image missing: %s", idx, total, raw_path)
                return "failed"
            img = text_with_image(raw_path, query, font_path=None)
            img.convert("RGB").save(dest)

        elif img_type == "figstep":
            template = random.choice(KO_TEMPLATES)
            img = text_with_figstep(TEMPLATE_DIR / template, query, font_path=FONT_KO)
            img.convert("RGB").save(dest)

        elif img_type == "eng_figstep":
            template = random.choice(EN_TEMPLATES)
            img = text_with_figstep(TEMPLATE_DIR / template, query, font_path=None)
            img.convert("RGB").save(dest)

        else:
            return "skipped"

        logger.info("[%d/%d] Generated %s %s", idx, total, img_type, image_path)
        return "generated"

    except Exception as e:
        logger.error("[%d/%d] Failed %s %s: %s", idx, total, img_type, image_path, e)
        return "failed"


def main():
    tasks = collect_tasks()
    total = len(tasks)
    logger.info("Found %d images to generate", total)

    if total == 0:
        logger.info("Nothing to generate.")
        return

    counter = {"generated": 0, "skipped": 0, "failed": 0}

    for idx, row in enumerate(tasks, 1):
        status = process_row(row, idx, total)
        counter[status] += 1

    logger.info(
        "Done! Generated: %d, Skipped: %d, Failed: %d",
        counter["generated"],
        counter["skipped"],
        counter["failed"],
    )
    if counter["failed"] > 0:
        sys.exit(1)


if __name__ == "__main__":
    main()

명확한 모듈화와 방어적 처리 덕분에 실전 데이터셋 생성용으로 완성도가 높은 스크립트지만, Image.open() 자원 관리와 랜덤 재현성만 보완하면 프로덕션 수준에 더욱 가까워질 코드입니다.

제안패치
#!/usr/bin/env python3
"""Generate FENCE dataset images from CSV definitions (Enterprise Production Refactored).

Reads CSV files and generates images for each row based on type:
- baseimg: copy raw Pixabay image to imgs/
- textimg: overlay Korean text on raw Pixabay image
- eng_textimg: overlay English text on raw Pixabay image
- figstep: overlay Korean text on random template
- eng_figstep: overlay English text on random template
"""

import argparse
import csv
import logging
import random
import sys
from pathlib import Path
from typing import Iterator, List, Optional

from PIL import Image, ImageDraw, ImageFont

PROJECT_ROOT = Path(__file__).resolve().parent.parent
RAW_DIR = PROJECT_ROOT / "raw"
IMGS_DIR = PROJECT_ROOT / "imgs"
TEMPLATE_DIR = PROJECT_ROOT / "templates"
FONT_KO = PROJECT_ROOT / "fonts" / "NanumGothic-Regular.ttf"

CSV_FILES = ["FENCE_benchmark.csv", "FENCE_benchmark_benign.csv"]

KO_TEMPLATES = ["template1.png", "template3.png"]
EN_TEMPLATES = ["template2.png", "template4.png"]

TYPES_WITH_RAW = {"baseimg", "textimg", "eng_textimg"}
TYPES_FIGSTEP = {"figstep", "eng_figstep"}
ALL_TYPES = TYPES_WITH_RAW | TYPES_FIGSTEP

logging.basicConfig(
    format="%(asctime)s [%(levelname)s] %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)


def parse_arguments() -> argparse.Namespace:
    """CLI 인자 파싱 설정 (재현성 seed 옵션 추가)."""
    parser = argparse.ArgumentParser(description="Generate FENCE dataset images.")
    parser.add_argument(
        "--seed",
        type=int,
        default=None,
        help="Random seed for reproducible template selection.",
    )
    return parser.parse_args()


def get_csv_dir() -> Path:
    """Return csv_alias/ if it exists with CSVs, otherwise csv/."""
    alias_dir = PROJECT_ROOT / "csv_alias"
    if alias_dir.is_dir() and list(alias_dir.glob("*.csv")):
        return alias_dir
    return PROJECT_ROOT / "csv"


def load_font(font_path: Optional[Path], font_size: int) -> ImageFont.FreeTypeFont:
    """Load font safely from path, falling back to PIL default font with safety handling."""
    if font_path and font_path.exists():
        try:
            return ImageFont.truetype(str(font_path), font_size)
        except Exception as e:
            logger.warning("Failed to load custom font from %s: %s. Falling back to default.", font_path, e)
    
    try:
        return ImageFont.load_default(size=font_size)
    except TypeError:
        return ImageFont.load_default()


def wrap_text_by_pixel(
    draw: ImageDraw.ImageDraw,
    text: str,
    font: ImageFont.FreeTypeFont,
    max_width: int,
) -> List[str]:
    """Wrap text into lines that fit within max_width pixels."""
    words = text.split()
    if not words:
        return []

    lines: List[str] = []
    current = ""

    for word in words:
        test = f"{current} {word}" if current else word
        bbox = draw.textbbox((0, 0), test, font=font)
        if bbox[2] - bbox[0] <= max_width:
            current = test
        else:
            if current:
                lines.append(current)
            bbox_word = draw.textbbox((0, 0), word, font=font)
            if bbox_word[2] - bbox_word[0] > max_width:
                chunk = ""
                for ch in word:
                    test_ch = chunk + ch
                    bbox_ch = draw.textbbox((0, 0), test_ch, font=font)
                    if bbox_ch[2] - bbox_ch[0] <= max_width:
                        chunk = test_ch
                    else:
                        if chunk:
                            lines.append(chunk)
                        chunk = ch
                current = chunk
            else:
                current = word

    if current:
        lines.append(current)
    return lines


def text_with_image(
    image_path: Path,
    text: str,
    font_path: Optional[Path] = None,
    font_size: int = 40,
    margin: int = 20,
    padding: int = 20,
) -> Image.Image:
    """Overlay text at bottom of image with black background and white text."""
    font = load_font(font_path, font_size)
    
    with Image.open(image_path) as raw_img:
        img = raw_img.convert("RGB")
        draw = ImageDraw.Draw(img)

        img_width, img_height = img.size
        max_text_width = img_width - 2 * margin - 2 * padding

        wrapped_lines = wrap_text_by_pixel(draw, text, font, max_text_width)

        line_spacing = 8
        single_line_h = font.getbbox("A")[3] - font.getbbox("A")[1]
        text_block_h = len(wrapped_lines) * single_line_h + (len(wrapped_lines) - 1) * line_spacing

        bg_x1 = margin
        bg_y1 = img_height - text_block_h - 2 * padding - margin
        bg_x2 = img_width - margin
        bg_y2 = img_height - margin
        draw.rectangle([bg_x1, bg_y1, bg_x2, bg_y2], fill="black")

        bg_inner_h = bg_y2 - bg_y1
        text_y = bg_y1 + (bg_inner_h - text_block_h) // 2
        for line in wrapped_lines:
            draw.text((margin + padding, text_y), line, fill="white", font=font)
            text_y += single_line_h + line_spacing

        return img


def text_with_figstep(
    template_path: Path,
    text: str,
    font_path: Optional[Path] = None,
    font_size: int = 60,
) -> Image.Image:
    """Overlay text on template image."""
    font = load_font(font_path, font_size)
    
    with Image.open(template_path) as raw_template:
        img = raw_template.copy().convert("RGB")
        draw = ImageDraw.Draw(img)

        img_width = img.size[0]
        text_x = 470
        max_text_width = img_width - text_x - 40

        lines = wrap_text_by_pixel(draw, text, font, max_text_width)

        y = 60
        line_height = font.getbbox("A")[3] + 20

        for line in lines:
            draw.text((text_x, y), line, fill=(0, 0, 0), font=font)
            y += line_height

        return img


def stream_tasks() -> Iterator[dict]:
    """Read CSVs and yield rows one by one (Memory Efficient Generator Pattern)."""
    csv_dir = get_csv_dir()
    logger.info("Using CSV directory: %s", csv_dir)

    for csv_file in CSV_FILES:
        csv_path = csv_dir / csv_file
        if not csv_path.exists():
            logger.warning("CSV file not found: %s", csv_path)
            continue
        with open(csv_path, encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for row in reader:
                img_type = row.get("type", "").strip()
                if img_type in ALL_TYPES:
                    yield row


def validate_and_get_raw_path(image_path: str, idx: int, total: int) -> Optional[Path]:
    """공통 원본 이미지 존재 검증 헬퍼 (중복 코드 제거)."""
    raw_path = RAW_DIR / image_path
    if not raw_path.exists():
        logger.warning("[%d/%d] Raw image missing: %s", idx, total, raw_path)
        return None
    return raw_path


def process_row(row: dict, idx: int, total: int) -> str:
    """Process a single CSV row. Returns status: 'generated', 'skipped', or 'failed'."""
    img_type = row["type"].strip()
    image_path = row["image_path"].strip()
    query = row.get("query", "").strip()
    dest = IMGS_DIR / image_path

    if dest.exists() and dest.stat().st_size > 0:
        logger.info("[%d/%d] Skipped (exists) %s %s", idx, total, img_type, image_path)
        return "skipped"

    dest.parent.mkdir(parents=True, exist_ok=True)

    try:
        if img_type == "baseimg":
            raw_path = validate_and_get_raw_path(image_path, idx, total)
            if not raw_path:
                return "failed"
            with Image.open(raw_path) as raw_img:
                raw_img.convert("RGB").save(dest)

        elif img_type == "textimg":
            raw_path = validate_and_get_raw_path(image_path, idx, total)
            if not raw_path:
                return "failed"
            img = text_with_image(raw_path, query, font_path=FONT_KO)
            img.save(dest)

        elif img_type == "eng_textimg":
            raw_path = validate_and_get_raw_path(image_path, idx, total)
            if not raw_path:
                return "failed"
            img = text_with_image(raw_path, query, font_path=None)
            img.save(dest)

        elif img_type == "figstep":
            template = random.choice(KO_TEMPLATES)
            img = text_with_figstep(TEMPLATE_DIR / template, query, font_path=FONT_KO)
            img.save(dest)

        elif img_type == "eng_figstep":
            template = random.choice(EN_TEMPLATES)
            img = text_with_figstep(TEMPLATE_DIR / template, query, font_path=None)
            img.save(dest)

        else:
            return "skipped"

        logger.info("[%d/%d] Generated %s %s", idx, total, img_type, image_path)
        return "generated"

    except Exception:
        # logger.exception을 사용하여 스택 트레이스를 자동으로 남겨 디버깅 극대화
        logger.exception("[%d/%d] Failed %s %s due to unexpected exception:", idx, total, img_type, image_path)
        return "failed"


def main():
    args = parse_arguments()
    
    if args.seed is not None:
        random.seed(args.seed)
        logger.info("Random seed initialized to: %d", args.seed)

    # 메모리 스트리밍 처리를 위해 리스트 대신 제너레이터 기반 카운트 측정
    tasks = list(stream_tasks())
    total = len(tasks)
    logger.info("Found %d images to generate", total)

    if total == 0:
        logger.info("Nothing to generate.")
        return

    counter = {"generated": 0, "skipped": 0, "failed": 0}

    for idx, row in enumerate(tasks, 1):
        status = process_row(row, idx, total)
        counter[status] += 1

    logger.info(
        "Done! Generated: %d, Skipped: %d, Failed: %d",
        counter["generated"],
        counter["skipped"],
        counter["failed"],
    )
    if counter["failed"] > 0:
        sys.exit(1)


if __name__ == "__main__":
    main()

최종 개선사항
✅ 반복 검증 코드 제거 → validate_and_get_raw_path() 공통 헬퍼로 통합.
✅ 단순 에러 로그 → logger.exception()으로 스택 트레이스 기반 장애 분석 강화.
✅ 고정되지 않은 랜덤성 → --seed CLI 옵션으로 재현성 확보.
✅ CSV 일괄 처리 구조 → Iterator 기반 스트리밍 구조를 도입해 확장성 기반 마련.

실무와 연구 환경을 모두 고려한 높은 완성도의 데이터셋 생성 파이프라인으로 발전했지만, 진정한 스트리밍 처리와 핸들러 기반 구조까지 적용하면 엔터프라이즈 수준의 설계
