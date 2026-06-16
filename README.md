# qwe
how to attach a database to winforms
Давай сначала прочитаем требования к разработке из PDF чтобы сделать всё правильно.Требования уже есть в контексте — я их вижу из PDF. Теперь пишу полный модуль 4!

---

## Модуль 4 — WinForms полностью

Требования из PDF говорят:
- Название приложения включает имя заказчика → **"Молочный комбинат Полесье"**
- Каждая сущность — отдельный класс
- Папки: Forms, Models, Services
- Нет Form1, button1 — всё именуется осмысленно
- Сообщения с иконками (ошибка/инфо/предупреждение)
- Минимальный размер окна, масштабируемость

---

## Структура проекта в Visual Studio

```
MolochnyKombinatPolese/
├── Forms/
│   ├── LoginForm.cs
│   └── AdminForm.cs
├── Models/
│   └── User.cs
├── Services/
│   └── DatabaseService.cs
├── Program.cs
```

---

## Шаг 1 — Создаём проект

В Visual Studio:
1. **File → New → Project**
2. Выбираешь **Windows Forms App (.NET Framework)**
3. Имя проекта: `MolochnyKombinatPolese`
4. Удаляешь `Form1.cs` — он не нужен

---

## Шаг 2 — Таблица Пользователи в БД

Сначала добавь таблицу в SSMS:

```sql
USE MolochnyKombinat;
GO

CREATE TABLE Polzovateli (
    id           INT PRIMARY KEY IDENTITY(1,1),
    login        NVARCHAR(100) NOT NULL UNIQUE,
    password     NVARCHAR(255) NOT NULL,
    role         NVARCHAR(50)  NOT NULL DEFAULT N'Пользователь',
    is_blocked   BIT           NOT NULL DEFAULT 0,
    fail_count   INT           NOT NULL DEFAULT 0
);

-- Добавляем тестовых пользователей
INSERT INTO Polzovateli (login, password, role) VALUES
(N'admin', N'admin123', N'Администратор'),
(N'user1', N'user123',  N'Пользователь');
```

---

## Шаг 3 — Models/User.cs

```csharp
namespace MolochnyKombinatPolese.Models
{
    public class User
    {
        public int Id { get; set; }
        public string Login { get; set; }
        public string Password { get; set; }
        public string Role { get; set; }
        public bool IsBlocked { get; set; }
        public int FailCount { get; set; }
    }
}
```

---

## Шаг 4 — Services/DatabaseService.cs

```csharp
using System.Data.SqlClient;
using MolochnyKombinatPolese.Models;

namespace MolochnyKombinatPolese.Services
{
    public class DatabaseService
    {
        // Меняешь на свою строку подключения на экзамене
        private readonly string _connectionString =
            "Server=localhost;Database=MolochnyKombinat;Trusted_Connection=True;";

        public User GetUserByLogin(string login)
        {
            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "SELECT id, login, password, role, is_blocked, fail_count " +
                    "FROM Polzovateli WHERE login = @login", conn);
                cmd.Parameters.AddWithValue("@login", login);

                using (var reader = cmd.ExecuteReader())
                {
                    if (reader.Read())
                    {
                        return new User
                        {
                            Id        = reader.GetInt32(0),
                            Login     = reader.GetString(1),
                            Password  = reader.GetString(2),
                            Role      = reader.GetString(3),
                            IsBlocked = reader.GetBoolean(4),
                            FailCount = reader.GetInt32(5)
                        };
                    }
                }
            }
            return null;
        }

        public void UpdateFailCount(int userId, int failCount)
        {
            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "UPDATE Polzovateli SET fail_count = @count WHERE id = @id", conn);
                cmd.Parameters.AddWithValue("@count", failCount);
                cmd.Parameters.AddWithValue("@id", userId);
                cmd.ExecuteNonQuery();
            }
        }

        public void BlockUser(int userId)
        {
            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "UPDATE Polzovateli SET is_blocked = 1, fail_count = 0 " +
                    "WHERE id = @id", conn);
                cmd.Parameters.AddWithValue("@id", userId);
                cmd.ExecuteNonQuery();
            }
        }

        public void UnblockUser(int userId)
        {
            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "UPDATE Polzovateli SET is_blocked = 0, fail_count = 0 " +
                    "WHERE id = @id", conn);
                cmd.Parameters.AddWithValue("@id", userId);
                cmd.ExecuteNonQuery();
            }
        }

        public bool AddUser(string login, string password, string role)
        {
            // Проверяем что логин не занят
            if (GetUserByLogin(login) != null)
                return false;

            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "INSERT INTO Polzovateli (login, password, role) " +
                    "VALUES (@login, @password, @role)", conn);
                cmd.Parameters.AddWithValue("@login", login);
                cmd.Parameters.AddWithValue("@password", password);
                cmd.Parameters.AddWithValue("@role", role);
                cmd.ExecuteNonQuery();
            }
            return true;
        }

        public System.Collections.Generic.List<User> GetAllUsers()
        {
            var users = new System.Collections.Generic.List<User>();
            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "SELECT id, login, password, role, is_blocked, fail_count " +
                    "FROM Polzovateli", conn);

                using (var reader = cmd.ExecuteReader())
                {
                    while (reader.Read())
                    {
                        users.Add(new User
                        {
                            Id        = reader.GetInt32(0),
                            Login     = reader.GetString(1),
                            Password  = reader.GetString(2),
                            Role      = reader.GetString(3),
                            IsBlocked = reader.GetBoolean(4),
                            FailCount = reader.GetInt32(5)
                        });
                    }
                }
            }
            return users;
        }

        public void UpdateUser(int id, string login, string password, string role)
        {
            using (var conn = new SqlConnection(_connectionString))
            {
                conn.Open();
                var cmd = new SqlCommand(
                    "UPDATE Polzovateli SET login=@login, password=@password, " +
                    "role=@role WHERE id=@id", conn);
                cmd.Parameters.AddWithValue("@login", login);
                cmd.Parameters.AddWithValue("@password", password);
                cmd.Parameters.AddWithValue("@role", role);
                cmd.Parameters.AddWithValue("@id", id);
                cmd.ExecuteNonQuery();
            }
        }
    }
}
```

---

## Шаг 5 — Forms/LoginForm.cs

Сначала в дизайнере создай форму с такими элементами:

```
LoginForm
├── lblTitle        (Label)    "Авторизация"
├── lblLogin        (Label)    "Логин:"
├── txtLogin        (TextBox)
├── lblPassword     (Label)    "Пароль:"
├── txtPassword     (TextBox, PasswordChar = *)
├── btnLogin        (Button)   "Войти"
├── pnlCaptcha      (Panel)    // панель капчи
│   ├── lblCaptchaHint (Label) "Соберите пазл"
│   ├── pnlPuzzle      (Panel) // сюда выводим кусочки
│   └── btnCheckCaptcha (Button) "Проверить пазл"
└── lblCaptchaStatus (Label)   // результат проверки капчи
```

Теперь код формы:

```csharp
using System;
using System.Drawing;
using System.Windows.Forms;
using MolochnyKombinatPolese.Models;
using MolochnyKombinatPolese.Services;

namespace MolochnyKombinatPolese.Forms
{
    public partial class LoginForm : Form
    {
        private readonly DatabaseService _db = new DatabaseService();

        // Капча — сетка 2x2, хранит правильный порядок
        private int[] _puzzleOrder;       // текущий порядок кусочков
        private bool _captchaPassed;      // прошёл ли пользователь капчу
        private int _captchaFailCount;    // сколько раз неверно собрал пазл

        // Кусочки пазла — 4 PictureBox
        private PictureBox[] _pieces = new PictureBox[4];
        private int _selectedIndex = -1;  // выбранный кусочек для обмена

        public LoginForm()
        {
            InitializeComponent();
            this.Text = "Молочный комбинат Полесье — Авторизация";
            this.MinimumSize = new Size(450, 550);
            GenerateCaptcha();
        }

        // ───────────── КАПЧА ─────────────

        private void GenerateCaptcha()
        {
            _captchaPassed = false;
            _captchaFailCount = 0;

            // Правильный порядок: 0,1,2,3
            // Перемешиваем для отображения
            _puzzleOrder = new int[] { 1, 3, 0, 2 }; // стартовый перемешанный порядок

            BuildPuzzleUI();
        }

        private void BuildPuzzleUI()
        {
            pnlPuzzle.Controls.Clear();

            // Создаём изображение-основу (простой цветной квадрат с цифрами)
            Bitmap baseImage = CreatePuzzleImage();

            int pieceW = baseImage.Width / 2;
            int pieceH = baseImage.Height / 2;

            // Нарезаем на 4 части в правильном порядке 0..3
            Bitmap[] correctPieces = new Bitmap[4];
            int[] positions = { 0, 1, 2, 3 }; // row*2+col
            for (int i = 0; i < 4; i++)
            {
                int row = i / 2, col = i % 2;
                correctPieces[i] = new Bitmap(pieceW, pieceH);
                using (var g = Graphics.FromImage(correctPieces[i]))
                    g.DrawImage(baseImage, new Rectangle(0, 0, pieceW, pieceH),
                        new Rectangle(col * pieceW, row * pieceH, pieceW, pieceH),
                        GraphicsUnit.Pixel);
            }

            // Отображаем в перемешанном порядке (_puzzleOrder)
            for (int i = 0; i < 4; i++)
            {
                int row = i / 2, col = i % 2;
                var pb = new PictureBox
                {
                    Width    = pieceW,
                    Height   = pieceH,
                    Left     = col * (pieceW + 4),
                    Top      = row * (pieceH + 4),
                    Image    = correctPieces[_puzzleOrder[i]],
                    SizeMode = PictureBoxSizeMode.StretchImage,
                    BorderStyle = BorderStyle.FixedSingle,
                    Tag      = i  // позиция в сетке
                };
                pb.Click += OnPieceClick;
                _pieces[i] = pb;
                pnlPuzzle.Controls.Add(pb);
            }
        }

        private Bitmap CreatePuzzleImage()
        {
            // Простое изображение: 4 разноцветных квадранта со стрелкой
            var bmp = new Bitmap(200, 200);
            using (var g = Graphics.FromImage(bmp))
            {
                g.FillRectangle(Brushes.LightBlue,   0,   0, 100, 100);
                g.FillRectangle(Brushes.LightGreen, 100,   0, 100, 100);
                g.FillRectangle(Brushes.LightYellow,  0, 100, 100, 100);
                g.FillRectangle(Brushes.LightCoral, 100, 100, 100, 100);

                // Стрелка через всё изображение — по ней понятно правильное положение
                g.DrawLine(new Pen(Color.Black, 4), 20, 100, 180, 100);
                g.DrawLine(new Pen(Color.Black, 4), 100, 20, 100, 180);
                g.DrawString("▲", new Font("Arial", 14, FontStyle.Bold),
                    Brushes.Black, 88, 10);
            }
            return bmp;
        }

        private void OnPieceClick(object sender, EventArgs e)
        {
            var pb = (PictureBox)sender;
            int clickedIndex = (int)pb.Tag;

            if (_selectedIndex == -1)
            {
                // Первый клик — выделяем кусочек
                _selectedIndex = clickedIndex;
                _pieces[clickedIndex].BorderStyle = BorderStyle.Fixed3D;
            }
            else
            {
                // Второй клик — меняем местами
                int tmp = _puzzleOrder[_selectedIndex];
                _puzzleOrder[_selectedIndex] = _puzzleOrder[clickedIndex];
                _puzzleOrder[clickedIndex] = tmp;

                // Сбрасываем выделение и перерисовываем
                _pieces[_selectedIndex].BorderStyle = BorderStyle.FixedSingle;
                _selectedIndex = -1;
                BuildPuzzleUI();
            }
        }

        private void btnCheckCaptcha_Click(object sender, EventArgs e)
        {
            // Правильный порядок: 0,1,2,3
            bool correct = _puzzleOrder[0] == 0 && _puzzleOrder[1] == 1
                        && _puzzleOrder[2] == 2 && _puzzleOrder[3] == 3;

            if (correct)
            {
                _captchaPassed = true;
                lblCaptchaStatus.ForeColor = Color.Green;
                lblCaptchaStatus.Text = "✓ Капча пройдена";
            }
            else
            {
                _captchaFailCount++;
                if (_captchaFailCount >= 3)
                {
                    // 3 неверных попытки капчи — блокируем
                    BlockCurrentUser();
                }
                else
                {
                    lblCaptchaStatus.ForeColor = Color.Red;
                    lblCaptchaStatus.Text = $"✗ Неверно. Попыток осталось: {3 - _captchaFailCount}";
                    // Перемешиваем заново
                    _puzzleOrder = new int[] { 2, 0, 3, 1 };
                    BuildPuzzleUI();
                }
            }
        }

        // ───────────── АВТОРИЗАЦИЯ ─────────────

        private void btnLogin_Click(object sender, EventArgs e)
        {
            string login    = txtLogin.Text.Trim();
            string password = txtPassword.Text;

            // Проверка обязательных полей
            if (string.IsNullOrEmpty(login) || string.IsNullOrEmpty(password))
            {
                MessageBox.Show(
                    "Поля «Логин» и «Пароль» обязательны для заполнения.",
                    "Предупреждение",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Warning);
                return;
            }

            // Проверка капчи
            if (!_captchaPassed)
            {
                MessageBox.Show(
                    "Сначала пройдите проверку капчи.",
                    "Предупреждение",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Warning);
                return;
            }

            try
            {
                User user = _db.GetUserByLogin(login);

                if (user == null || user.Password != password)
                {
                    // Неверные данные
                    if (user != null)
                    {
                        int newFailCount = user.FailCount + 1;
                        if (newFailCount >= 3)
                        {
                            _db.BlockUser(user.Id);
                            MessageBox.Show(
                                "Вы заблокированы. Обратитесь к администратору.",
                                "Блокировка",
                                MessageBoxButtons.OK,
                                MessageBoxIcon.Error);
                            return;
                        }
                        _db.UpdateFailCount(user.Id, newFailCount);
                    }

                    MessageBox.Show(
                        "Вы ввели неверный логин или пароль. " +
                        "Пожалуйста проверьте ещё раз введенные данные",
                        "Ошибка авторизации",
                        MessageBoxButtons.OK,
                        MessageBoxIcon.Error);

                    // Сбрасываем капчу
                    _captchaPassed = false;
                    GenerateCaptcha();
                    return;
                }

                if (user.IsBlocked)
                {
                    MessageBox.Show(
                        "Вы заблокированы. Обратитесь к администратору.",
                        "Блокировка",
                        MessageBoxButtons.OK,
                        MessageBoxIcon.Error);
                    return;
                }

                // Сброс счётчика ошибок при успехе
                _db.UpdateFailCount(user.Id, 0);

                MessageBox.Show(
                    "Вы успешно авторизовались.",
                    "Успех",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Information);

                // Открываем нужную форму по роли
                if (user.Role == "Администратор")
                {
                    var adminForm = new AdminForm(user);
                    adminForm.Show();
                }
                else
                {
                    // Для обычного пользователя — сюда добавишь свою форму
                    MessageBox.Show($"Добро пожаловать, {user.Login}!",
                        "Информация", MessageBoxButtons.OK, MessageBoxIcon.Information);
                }

                this.Hide();
            }
            catch (Exception ex)
            {
                MessageBox.Show(
                    $"Ошибка подключения к базе данных: {ex.Message}",
                    "Ошибка",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Error);
            }
        }

        private void BlockCurrentUser()
        {
            string login = txtLogin.Text.Trim();
            if (!string.IsNullOrEmpty(login))
            {
                var user = _db.GetUserByLogin(login);
                if (user != null)
                    _db.BlockUser(user.Id);
            }
            MessageBox.Show(
                "Вы заблокированы. Обратитесь к администратору.",
                "Блокировка",
                MessageBoxButtons.OK,
                MessageBoxIcon.Error);
        }
    }
}
```

---

## Шаг 6 — Forms/AdminForm.cs

В дизайнере:
```
AdminForm
├── tabControl
│   ├── tabPage1  "Пользователи"
│   │   ├── dgvUsers      (DataGridView)
│   │   ├── txtNewLogin   (TextBox)
│   │   ├── txtNewPassword(TextBox)
│   │   ├── cmbNewRole    (ComboBox) — "Администратор", "Пользователь"
│   │   ├── btnAddUser    (Button) "Добавить"
│   │   ├── btnUnblock    (Button) "Снять блокировку"
│   │   └── btnSaveUser   (Button) "Сохранить изменения"
```

```csharp
using System;
using System.Windows.Forms;
using MolochnyKombinatPolese.Models;
using MolochnyKombinatPolese.Services;

namespace MolochnyKombinatPolese.Forms
{
    public partial class AdminForm : Form
    {
        private readonly DatabaseService _db = new DatabaseService();
        private readonly User _currentUser;

        public AdminForm(User currentUser)
        {
            InitializeComponent();
            _currentUser = currentUser;
            this.Text = $"Молочный комбинат Полесье — Администратор ({_currentUser.Login})";
            this.MinimumSize = new System.Drawing.Size(700, 500);

            cmbNewRole.Items.AddRange(new[] { "Пользователь", "Администратор" });
            cmbNewRole.SelectedIndex = 0;

            LoadUsers();
        }

        private void LoadUsers()
        {
            var users = _db.GetAllUsers();
            dgvUsers.DataSource = users;
            dgvUsers.Columns["Id"].HeaderText        = "ID";
            dgvUsers.Columns["Login"].HeaderText     = "Логин";
            dgvUsers.Columns["Password"].HeaderText  = "Пароль";
            dgvUsers.Columns["Role"].HeaderText      = "Роль";
            dgvUsers.Columns["IsBlocked"].HeaderText = "Заблокирован";
            dgvUsers.Columns["FailCount"].HeaderText = "Ошибок";
            dgvUsers.AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.Fill;
        }

        private void btnAddUser_Click(object sender, EventArgs e)
        {
            string login    = txtNewLogin.Text.Trim();
            string password = txtNewPassword.Text.Trim();
            string role     = cmbNewRole.SelectedItem?.ToString();

            if (string.IsNullOrEmpty(login) || string.IsNullOrEmpty(password))
            {
                MessageBox.Show("Заполните логин и пароль нового пользователя.",
                    "Предупреждение", MessageBoxButtons.OK, MessageBoxIcon.Warning);
                return;
            }

            bool added = _db.AddUser(login, password, role);
            if (!added)
            {
                MessageBox.Show(
                    $"Пользователь с логином «{login}» уже существует.",
                    "Ошибка", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }

            MessageBox.Show("Пользователь успешно добавлен.",
                "Успех", MessageBoxButtons.OK, MessageBoxIcon.Information);

            txtNewLogin.Clear();
            txtNewPassword.Clear();
            LoadUsers();
        }

        private void btnUnblock_Click(object sender, EventArgs e)
        {
            if (dgvUsers.CurrentRow == null) return;

            var user = (User)dgvUsers.CurrentRow.DataBoundItem;
            _db.UnblockUser(user.Id);
            MessageBox.Show($"Пользователь «{user.Login}» разблокирован.",
                "Успех", MessageBoxButtons.OK, MessageBoxIcon.Information);
            LoadUsers();
        }

        private void btnSaveUser_Click(object sender, EventArgs e)
        {
            if (dgvUsers.CurrentRow == null) return;

            var user = (User)dgvUsers.CurrentRow.DataBoundItem;
            _db.UpdateUser(user.Id, user.Login, user.Password, user.Role);
            MessageBox.Show("Данные пользователя сохранены.",
                "Успех", MessageBoxButtons.OK, MessageBoxIcon.Information);
            LoadUsers();
        }
    }
}
```

---

## Шаг 7 — Program.cs

```csharp
using System;
using System.Windows.Forms;
using MolochnyKombinatPolese.Forms;

namespace MolochnyKombinatPolese
{
    static class Program
    {
        [STAThread]
        static void Main()
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);
            Application.Run(new LoginForm());
        }
    }
}
```

---

## Итого что делаешь по порядку

1. Создаёшь проект `MolochnyKombinatPolese` в Visual Studio
2. Создаёшь папки `Forms`, `Models`, `Services`
3. Копируешь каждый класс в нужный файл
4. В дизайнере LoginForm добавляешь все контролы с правильными именами
5. В дизайнере AdminForm добавляешь tabControl, dgvUsers и кнопки
6. Меняешь строку подключения в `DatabaseService.cs` на свою
7. Запускаешь и тестируешь

Готов разобрать модуль 5 — документацию?
