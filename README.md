import sys
import os
import math
import random
import pygame
import arabic_reshaper
from bidi.algorithm import get_display

pygame.init()
pygame.mixer.init()

WIDTH, HEIGHT = 1000, 650
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("نبض وطن | Nabd Watan — جامعة نجران")

DARK_GREEN = (2, 38, 28)
EMERALD = (5, 68, 48)
GOLD = (212, 175, 55)
LIGHT_GOLD = (250, 225, 130)
GLOW_GOLD = (255, 240, 170)
WHITE = (248, 249, 250)
TEAL_CARD = (8, 55, 42)
ACCENT_GREEN = (30, 120, 80)
CORRECT_GREEN = (40, 140, 60)
WRONG_RED = (180, 45, 45)

clock = pygame.time.Clock()

current_screen = "WELCOME"
target_screen = "WELCOME"
fade_alpha = 0
is_fading = False

quiz_score = 0
current_q_index = 0
selected_option = None
show_feedback = False
feedback_timer = 0
selected_section = "HERITAGE"

particles = [
    {
        "x": random.randint(0, WIDTH),
        "y": random.randint(0, HEIGHT),
        "radius": random.uniform(1.5, 4.0),
        "speed": random.uniform(0.3, 1.0),
        "alpha": random.randint(60, 160)
    }
    for _ in range(50)
]

def get_arabic_font(size, bold=False):
    possible_paths = [
        "Amiri-Regular.ttf",
        os.path.join("assets", "fonts", "font.ttf"),
        os.path.join("assets", "fonts", "Amiri-Regular.ttf")
    ]
    for font_path in possible_paths:
        if os.path.exists(font_path):
            return pygame.font.Font(font_path, size)
    
    for font_name in ["tahoma", "segoeui", "arial"]:
        f = pygame.font.SysFont(font_name, size, bold=bold)
        if f:
            return f
    return pygame.font.SysFont(None, size)

def render_arabic(text, size, color, bold=False):
    font = get_arabic_font(size, bold)
    reshaped_text = arabic_reshaper.reshape(text)
    bidi_text = get_display(reshaped_text)
    return font.render(bidi_text, True, color)

def draw_geometric_background():
    screen.fill(DARK_GREEN)
    t = pygame.time.get_ticks() * 0.001
    
    center_x, center_y = WIDTH // 2, HEIGHT // 2
    for i in range(12):
        angle = math.radians(i * 30 + math.sin(t * 0.5) * 5)
        x1 = center_x + math.cos(angle) * 80
        y1 = center_y + math.sin(angle) * 80
        x2 = center_x + math.cos(angle) * 450
        y2 = center_y + math.sin(angle) * 450
        pygame.draw.line(screen, (10, 50, 38), (x1, y1), (x2, y2), 1)

    for p in particles:
        p["y"] -= p["speed"]
        if p["y"] < -10:
            p["y"] = HEIGHT + 10
            p["x"] = random.randint(0, WIDTH)

        s = pygame.Surface((int(p["radius"] * 2), int(p["radius"] * 2)), pygame.SRCALPHA)
        pygame.draw.circle(s, (212, 175, 55, p["alpha"]), (int(p["radius"]), int(p["radius"])), int(p["radius"]))
        screen.blit(s, (p["x"], p["y"]))

def draw_bottom_visualizer():
    t = pygame.time.get_ticks() * 0.005
    for x in range(0, WIDTH, 12):
        height = math.sin(t + x * 0.03) * 12 + 15
        pygame.draw.line(screen, (212, 175, 55, 60), (x, HEIGHT - 5), (x, HEIGHT - 5 - height), 2)

def trigger_fade_to(next_screen):
    global is_fading, target_screen
    is_fading = True
    target_screen = next_screen

SECTIONS = {
    "HERITAGE": {
        "title": "إرثنا — Our Roots",
        "items": [
            ("الدرعية التاريخية", "عاصمة الدولة السعودية الأولى ورمز التراث التليد ومسيرة المجد."),
            ("العرضة السعودية", "رقصة النصر والاعتزاز بالهوية المسجلة عالمياً لدى منظمة اليونسكو."),
            ("الكرم والأصالة", "تقاليد ضيافة عربية فريدة تناقلتها الأجيال لترسخ القيم الأصيلة.")
        ]
    },
    "LAND": {
        "title": "أرضنا — Our Land",
        "items": [
            ("قمم نجران وعسير", "طبيعة جبلية ساحرة وتنوع جغرافي يجسد جمال تضاريس الوطن."),
            ("سواحل البحر الأحمر", "شواطئ فيروزية ومشروعات سياحية عالمية تحمي التنوع البيئي."),
            ("كثبان النفود والربع الخالي", "رمال ذهبية تحكي قصة صمود الإنسان وارتباطه الوثيق بأرضه.")
        ]
    },
    "FUTURE": {
        "title": "رؤيتنا — Our Future",
        "items": [
            ("مدينة نيوم", "نموذج عالمي لمعيشة المستقبل القائمة على الابتكار والطاقة النظيفة."),
            ("السعودية الخضراء", "مبادرات بيئية لاستزراع الملايين من الأشجار وتحقيق الحياد الصفري."),
            ("تمكين الابتكار", "اقتصاد معرفي يرتكز على إبداع وموهبة العقول الوطنية الشابة.")
        ]
    },
    "VALUES": {
        "title": "قيمنا — Our Values",
        "items": [
            ("طموح عنان السماء", "رؤية ثاقبة وإرادة قوية ترتقي بالوطن لمصاف الدول المتقدمة."),
            ("التلاحم والوحدة", "نسيج مجتمعي متماسك ملتف حول قيادته ومبادئه الوطنية."),
            ("الاعتزاز بالهوية", "افتخار ثابت بالإرث الحضاري والثقافي العريق للمملكة.")
        ]
    }
}

QUIZ_DATA = [
    {
        "q": "ما هي عاصمة الدولة السعودية الأولى؟",
        "options": ["الرياض", "الدرعية", "جدة"],
        "answer": 1
    },
    {
        "q": "ما الشعار الوطني لليوم الوطني السعودي؟",
        "options": ["نحلم ونحقق", "همة حتى القمة", "معاً لنرتقي"],
        "answer": 0
    },
    {
        "q": "ما اسم المدينة المستقبلية الضخمة القائمة على الطاقة النظيفة؟",
        "options": ["العلا", "نيوم", "القدية"],
        "answer": 1
    },
    {
        "q": "أين تقع جامعة نجران المنظمة لمسابقة وطن يبرمج؟",
        "options": ["منطقة نجران", "منطقة الرياض", "منطقة مكة المكرمة"],
        "answer": 0
    }
]


def draw_welcome_screen(mouse_pos):
    draw_geometric_background()
    draw_bottom_visualizer()

    t = pygame.time.get_ticks() * 0.003
    glow_val = int(200 + math.sin(t) * 55)
    title_gold = (glow_val, int(glow_val * 0.8), 50)

    font_en = pygame.font.SysFont("arial", 54, bold=True)
    title_en = font_en.render("NABD WATAN", True, WHITE)
    screen.blit(title_en, title_en.get_rect(center=(WIDTH // 2, 180)))

    title_ar = render_arabic("نبض وطن", 52, title_gold, bold=True)
    screen.blit(title_ar, title_ar.get_rect(center=(WIDTH // 2, 255)))

    sub_txt = render_arabic("رحلة تفاعلية في قصة وطن", 22, WHITE)
    screen.blit(sub_txt, sub_txt.get_rect(center=(WIDTH // 2, 320)))

    base_btn_rect = pygame.Rect(350, 400, 300, 65)
    is_hovered = base_btn_rect.collidepoint(mouse_pos)
    
    draw_rect = base_btn_rect.inflate(10, 6) if is_hovered else base_btn_rect

    pygame.draw.rect(screen, LIGHT_GOLD if is_hovered else GOLD, draw_rect, border_radius=18)
    pygame.draw.rect(screen, GLOW_GOLD if is_hovered else GOLD, draw_rect, width=3, border_radius=18)

    btn_txt = render_arabic("ابدأ الرحلة", 26, DARK_GREEN, bold=True)
    screen.blit(btn_txt, btn_txt.get_rect(center=draw_rect.center))

    foot_txt = render_arabic("وطن يبرمج • جامعة نجران", 17, WHITE)
    screen.blit(foot_txt, foot_txt.get_rect(center=(WIDTH // 2, 540)))

    return base_btn_rect

def draw_menu_screen(mouse_pos):
    draw_geometric_background()
    draw_bottom_visualizer()

    header = render_arabic("اختر محطة الرحلة", 38, GOLD, bold=True)
    screen.blit(header, header.get_rect(center=(WIDTH // 2, 60)))

    cards = [
        {"id": "HERITAGE", "title_ar": "إرثنا", "sub": "Our Roots", "rect": pygame.Rect(180, 125, 300, 140)},
        {"id": "LAND", "title_ar": "أرضنا", "sub": "Our Land", "rect": pygame.Rect(520, 125, 300, 140)},
        {"id": "FUTURE", "title_ar": "رؤيتنا", "sub": "Our Future", "rect": pygame.Rect(180, 285, 300, 140)},
        {"id": "VALUES", "title_ar": "قيمنا", "sub": "Our Values", "rect": pygame.Rect(520, 285, 300, 140)},
    ]

    card_clicked = None
    for card in cards:
        r = card["rect"]
        is_hovered = r.collidepoint(mouse_pos)
        
        draw_rect = r.inflate(8, 8) if is_hovered else r

        pygame.draw.rect(screen, TEAL_CARD, draw_rect, border_radius=14)
        pygame.draw.rect(screen, LIGHT_GOLD if is_hovered else GOLD, draw_rect, width=2, border_radius=14)

        txt_ar = render_arabic(card["title_ar"], 32, WHITE, bold=True)
        txt_en = pygame.font.SysFont("arial", 18).render(card["sub"], True, GOLD)

        screen.blit(txt_ar, txt_ar.get_rect(center=(draw_rect.centerx, draw_rect.centery - 15)))
        screen.blit(txt_en, txt_en.get_rect(center=(draw_rect.centerx, draw_rect.centery + 20)))

        if is_hovered:
            card_clicked = card["id"]

    quiz_btn_rect = pygame.Rect(350, 455, 300, 55)
    is_q_hovered = quiz_btn_rect.collidepoint(mouse_pos)
    draw_q_rect = quiz_btn_rect.inflate(8, 4) if is_q_hovered else quiz_btn_rect

    pygame.draw.rect(screen, ACCENT_GREEN if is_q_hovered else EMERALD, draw_q_rect, border_radius=12)
    pygame.draw.rect(screen, GOLD, draw_q_rect, width=2, border_radius=12)
    
    q_txt = render_arabic("تحدي وطن", 22, WHITE, bold=True)
    screen.blit(q_txt, q_txt.get_rect(center=draw_q_rect.center))

    back_btn_rect = pygame.Rect(400, 535, 200, 45)
    is_b_hovered = back_btn_rect.collidepoint(mouse_pos)
    pygame.draw.rect(screen, LIGHT_GOLD if is_b_hovered else GOLD, back_btn_rect, border_radius=10)
    back_txt = render_arabic("رجوع", 18, DARK_GREEN, bold=True)
    screen.blit(back_txt, back_txt.get_rect(center=back_btn_rect.center))

    return card_clicked, quiz_btn_rect, back_btn_rect

def draw_content_screen(section_key, mouse_pos):
    draw_geometric_background()
    draw_bottom_visualizer()

    data = SECTIONS[section_key]

    header = render_arabic(data["title"], 36, GOLD, bold=True)
    screen.blit(header, header.get_rect(center=(WIDTH // 2, 55)))

    for i, (title, desc) in enumerate(data["items"]):
        card_rect = pygame.Rect(120, 115 + (i * 125), 760, 105)
        
        is_hovered = card_rect.collidepoint(mouse_pos)
        draw_card = card_rect.inflate(6, 4) if is_hovered else card_rect

        pygame.draw.rect(screen, TEAL_CARD, draw_card, border_radius=14)
        pygame.draw.rect(screen, LIGHT_GOLD if is_hovered else GOLD, draw_card, width=2 if is_hovered else 1, border_radius=14)

        t_txt = render_arabic(title, 24, GOLD, bold=True)
        screen.blit(t_txt, t_txt.get_rect(center=(draw_card.centerx, draw_card.top + 30)))

        d_txt = render_arabic(desc, 18, WHITE)
        screen.blit(d_txt, d_txt.get_rect(center=(draw_card.centerx, draw_card.top + 65)))

    back_btn_rect = pygame.Rect(400, 520, 200, 45)
    is_b_hovered = back_btn_rect.collidepoint(mouse_pos)
    pygame.draw.rect(screen, LIGHT_GOLD if is_b_hovered else GOLD, back_btn_rect, border_radius=10)
    back_txt = render_arabic("القائمة الرئيسية", 20, DARK_GREEN, bold=True)
    screen.blit(back_txt, back_txt.get_rect(center=back_btn_rect.center))

    return back_btn_rect

def draw_quiz_screen(mouse_pos):
    global current_q_index, quiz_score, show_feedback, feedback_timer, selected_option
    draw_geometric_background()
    draw_bottom_visualizer()

    if current_q_index >= len(QUIZ_DATA):
        res_title = render_arabic("اكتمل التحدي بنجاح", 38, GOLD, bold=True)
        screen.blit(res_title, res_title.get_rect(center=(WIDTH // 2, 160)))

        score_str = f"نتيجتك: {quiz_score} من {len(QUIZ_DATA)}"
        score_txt = render_arabic(score_str, 30, WHITE, bold=True)
        screen.blit(score_txt, score_txt.get_rect(center=(WIDTH // 2, 230)))

        badge = "وسام: خبير بالوطن" if quiz_score >= 3 else "وسام: شغوف بالوطن"
        badge_txt = render_arabic(badge, 24, GOLD, bold=True)
        screen.blit(badge_txt, badge_txt.get_rect(center=(WIDTH // 2, 290)))

        msg_txt = render_arabic("أنت جزء ملهم من قصة وطن عظيم", 22, LIGHT_GOLD)
        screen.blit(msg_txt, msg_txt.get_rect(center=(WIDTH // 2, 360)))

        back_btn_rect = pygame.Rect(400, 460, 200, 50)
        is_hovered = back_btn_rect.collidepoint(mouse_pos)
        pygame.draw.rect(screen, LIGHT_GOLD if is_hovered else GOLD, back_btn_rect, border_radius=10)
        back_txt = render_arabic("الرئيسية", 22, DARK_GREEN, bold=True)
        screen.blit(back_txt, back_txt.get_rect(center=back_btn_rect.center))

        return None, back_btn_rect

    q_data = QUIZ_DATA[current_q_index]
    
    q_num_str = f"السؤال {current_q_index + 1} من {len(QUIZ_DATA)}"
    q_num = render_arabic(q_num_str, 20, GOLD)
    screen.blit(q_num, q_num.get_rect(center=(WIDTH // 2, 55)))

    q_txt = render_arabic(q_data["q"], 26, WHITE, bold=True)
    screen.blit(q_txt, q_txt.get_rect(center=(WIDTH // 2, 125)))

    opt_rects = []
    for i, opt in enumerate(q_data["options"]):
        rect = pygame.Rect(250, 200 + (i * 90), 500, 65)
        
        card_bg = TEAL_CARD
        border_col = GOLD

        if show_feedback and selected_option == i:
            if i == q_data["answer"]:
                card_bg = CORRECT_GREEN
                border_col = WHITE
            else:
                card_bg = WRONG_RED
                border_col = WHITE
        elif rect.collidepoint(mouse_pos) and not show_feedback:
            border_col = LIGHT_GOLD

        pygame.draw.rect(screen, card_bg, rect, border_radius=10)
        pygame.draw.rect(screen, border_col, rect, width=2, border_radius=10)

        opt_txt = render_arabic(opt, 22, WHITE)
        screen.blit(opt_txt, opt_txt.get_rect(center=rect.center))
        opt_rects.append((rect, i))

    return opt_rects, None


running = True
start_btn = None
menu_card_id, quiz_btn, menu_back_btn = None, None, None
content_back_btn = None
quiz_opts, quiz_finish_back = None, None

while running:
    mouse_pos = pygame.mouse.get_pos()

    if current_screen == "WELCOME":
        start_btn = draw_welcome_screen(mouse_pos)
    elif current_screen == "MENU":
        menu_card_id, quiz_btn, menu_back_btn = draw_menu_screen(mouse_pos)
    elif current_screen == "CONTENT":
        content_back_btn = draw_content_screen(selected_section, mouse_pos)
    elif current_screen == "QUIZ":
        quiz_opts, quiz_finish_back = draw_quiz_screen(mouse_pos)

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

        if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1 and not is_fading:
            if current_screen == "WELCOME":
                if start_btn and start_btn.collidepoint(mouse_pos):
                    trigger_fade_to("MENU")

            elif current_screen == "MENU":
                if menu_card_id:
                    selected_section = menu_card_id
                    trigger_fade_to("CONTENT")
                elif quiz_btn and quiz_btn.collidepoint(mouse_pos):
                    trigger_fade_to("QUIZ")
                    current_q_index = 0
                    quiz_score = 0
                    show_feedback = False
                elif menu_back_btn and menu_back_btn.collidepoint(mouse_pos):
                    trigger_fade_to("WELCOME")

            elif current_screen == "CONTENT":
                if content_back_btn and content_back_btn.collidepoint(mouse_pos):
                    trigger_fade_to("MENU")

            elif current_screen == "QUIZ":
                if quiz_opts and not show_feedback:
                    for rect, idx in quiz_opts:
                        if rect.collidepoint(mouse_pos):
                            selected_option = idx
                            show_feedback = True
                            feedback_timer = pygame.time.get_ticks()
                            if idx == QUIZ_DATA[current_q_index]["answer"]:
                                quiz_score += 1

                elif quiz_finish_back and quiz_finish_back.collidepoint(mouse_pos):
                    trigger_fade_to("MENU")

    if current_screen == "QUIZ" and show_feedback:
        if pygame.time.get_ticks() - feedback_timer > 900:
            show_feedback = False
            current_q_index += 1

    if is_fading:
        fade_alpha += 25
        fade_surface = pygame.Surface((WIDTH, HEIGHT))
        fade_surface.fill((0, 0, 0))
        fade_surface.set_alpha(fade_alpha)
        screen.blit(fade_surface, (0, 0))
        
        if fade_alpha >= 255:
            current_screen = target_screen
            is_fading = False
            fade_alpha = 0

    pygame.display.flip()
    clock.tick(60)

pygame.quit()
sys.exit()
