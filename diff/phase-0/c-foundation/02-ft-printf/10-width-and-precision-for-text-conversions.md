# 문자열 변환의 너비와 정밀도

## `feat(text): 문자·문자열·퍼센트 변환 추가`

diff --git a/Makefile b/Makefile
index 210a17a..fb35e4e 100644
--- a/Makefile
+++ b/Makefile
@@ -9,7 +9,9 @@ RM := rm -f
 
 SRC := src/ft_printf.c \
 	src/ft_output.c \
-	src/ft_parse.c
+	src/ft_parse.c \
+	src/ft_dispatch.c \
+	src/ft_text.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_dispatch.c b/src/ft_dispatch.c
new file mode 100644
index 0000000..dede10c
--- /dev/null
+++ b/src/ft_dispatch.c
@@ -0,0 +1,16 @@
+#include "ft_printf_internal.h"
+
+int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
+{
+	if (fmt->spec == 'c')
+		return (ft_printf_print_char(ctx, fmt, va_arg(*args, int)));
+	if (fmt->spec == 's')
+		return (ft_printf_print_string(ctx, fmt, va_arg(*args, char *)));
+	if (fmt->spec == '%')
+		return (ft_printf_print_percent(ctx, fmt));
+	if (ft_printf_putchar(ctx, '%') < 0)
+		return (-1);
+	if (fmt->spec)
+		return (ft_printf_putchar(ctx, fmt->spec));
+	return (0);
+}
diff --git a/src/ft_printf.c b/src/ft_printf.c
index 309fd28..be2dedd 100644
--- a/src/ft_printf.c
+++ b/src/ft_printf.c
@@ -1,7 +1,5 @@
 #include "ft_printf_internal.h"
 
-#include <stdarg.h>
-
 int	ft_printf(const char *format, ...)
 {
 	va_list	args;
@@ -29,10 +27,7 @@ int	ft_printf(const char *format, ...)
 				ctx.error = 1;
 				break ;
 			}
-			if (ft_printf_putchar(&ctx, '%') < 0)
-				break ;
-			if (fmt.spec != '\0' && fmt.spec != '%'
-				&& ft_printf_putchar(&ctx, fmt.spec) < 0)
+			if (ft_printf_dispatch(&ctx, &fmt, &args) < 0)
 				break ;
 		}
 		else
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index e4c6e47..70695ad 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -3,6 +3,7 @@
 
 # include "ft_printf.h"
 
+# include <stdarg.h>
 # include <stddef.h>
 
 # define FT_FLAG_LEFT 1
@@ -32,5 +33,10 @@ int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
 void		ft_printf_init_format(t_format *fmt);
 const char	*ft_printf_parse(const char *format, t_format *fmt);
+int			ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args);
+int			ft_printf_print_char(t_printf *ctx, t_format *fmt, int c);
+int			ft_printf_print_string(t_printf *ctx, t_format *fmt,
+				const char *string);
+int			ft_printf_print_percent(t_printf *ctx, t_format *fmt);
 
 #endif
diff --git a/src/ft_text.c b/src/ft_text.c
new file mode 100644
index 0000000..1455aa6
--- /dev/null
+++ b/src/ft_text.c
@@ -0,0 +1,31 @@
+#include "ft_printf_internal.h"
+
+static size_t	ft_local_strlen(const char *string)
+{
+	size_t	length;
+
+	length = 0;
+	while (string[length])
+		length++;
+	return (length);
+}
+
+int	ft_printf_print_char(t_printf *ctx, t_format *fmt, int c)
+{
+	(void)fmt;
+	return (ft_printf_putchar(ctx, (char)c));
+}
+
+int	ft_printf_print_string(t_printf *ctx, t_format *fmt, const char *string)
+{
+	(void)fmt;
+	if (string == 0)
+		string = "(null)";
+	return (ft_printf_write(ctx, string, ft_local_strlen(string)));
+}
+
+int	ft_printf_print_percent(t_printf *ctx, t_format *fmt)
+{
+	(void)fmt;
+	return (ft_printf_putchar(ctx, '%'));
+}


## `feat(text): 문자·문자열·퍼센트 너비와 정렬 적용`

diff --git a/src/ft_output.c b/src/ft_output.c
index de1d83a..1a214c3 100644
--- a/src/ft_output.c
+++ b/src/ft_output.c
@@ -40,3 +40,14 @@ int	ft_printf_putchar(t_printf *ctx, char c)
 {
 	return (ft_printf_write(ctx, &c, 1));
 }
+
+int	ft_printf_putnchar(t_printf *ctx, char c, int length)
+{
+	while (length > 0)
+	{
+		if (ft_printf_putchar(ctx, c) < 0)
+			return (-1);
+		length--;
+	}
+	return (0);
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 4bc3bb9..9192c92 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -31,6 +31,7 @@ typedef struct s_format
 void		ft_printf_init(t_printf *ctx, int fd);
 int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
+int			ft_printf_putnchar(t_printf *ctx, char c, int length);
 void		ft_printf_init_format(t_format *fmt);
 const char	*ft_printf_parse(const char *format, t_format *fmt);
 int			ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args);
diff --git a/src/ft_text.c b/src/ft_text.c
index 1455aa6..36e331d 100644
--- a/src/ft_text.c
+++ b/src/ft_text.c
@@ -12,20 +12,52 @@ static size_t	ft_local_strlen(const char *string)
 
 int	ft_printf_print_char(t_printf *ctx, t_format *fmt, int c)
 {
-	(void)fmt;
-	return (ft_printf_putchar(ctx, (char)c));
+	int	padding;
+
+	padding = fmt->width - 1;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_putchar(ctx, (char)c) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
 }
 
 int	ft_printf_print_string(t_printf *ctx, t_format *fmt, const char *string)
 {
-	(void)fmt;
+	size_t	length;
+	int		padding;
+
 	if (string == 0)
 		string = "(null)";
-	return (ft_printf_write(ctx, string, ft_local_strlen(string)));
+	length = ft_local_strlen(string);
+	padding = fmt->width - (int)length;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, string, length) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
 }
 
 int	ft_printf_print_percent(t_printf *ctx, t_format *fmt)
 {
-	(void)fmt;
-	return (ft_printf_putchar(ctx, '%'));
+	int	padding;
+
+	padding = fmt->width - 1;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_putchar(ctx, '%') < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
 }


## `feat(text): 문자열 정밀도와 퍼센트 0 채움 적용`

diff --git a/src/ft_text.c b/src/ft_text.c
index 36e331d..bbf5323 100644
--- a/src/ft_text.c
+++ b/src/ft_text.c
@@ -34,6 +34,8 @@ int	ft_printf_print_string(t_printf *ctx, t_format *fmt, const char *string)
 	if (string == 0)
 		string = "(null)";
 	length = ft_local_strlen(string);
+	if (fmt->has_precision && fmt->precision < (int)length)
+		length = (size_t)fmt->precision;
 	padding = fmt->width - (int)length;
 	if (!(fmt->flags & FT_FLAG_LEFT)
 		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
@@ -49,10 +51,14 @@ int	ft_printf_print_string(t_printf *ctx, t_format *fmt, const char *string)
 int	ft_printf_print_percent(t_printf *ctx, t_format *fmt)
 {
 	int	padding;
+	char	pad_char;
 
 	padding = fmt->width - 1;
+	pad_char = ' ';
+	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT))
+		pad_char = '0';
 	if (!(fmt->flags & FT_FLAG_LEFT)
-		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		&& ft_printf_putnchar(ctx, pad_char, padding) < 0)
 		return (-1);
 	if (ft_printf_putchar(ctx, '%') < 0)
 		return (-1);


## `fix(text): 문자열 정밀도 범위까지만 읽기`

diff --git a/src/ft_text.c b/src/ft_text.c
index bbf5323..3c3e923 100644
--- a/src/ft_text.c
+++ b/src/ft_text.c
@@ -1,11 +1,12 @@
 #include "ft_printf_internal.h"
 
-static size_t	ft_local_strlen(const char *string)
+static size_t	ft_local_strlen(const char *string, t_format *fmt)
 {
 	size_t	length;
 
 	length = 0;
-	while (string[length])
+	while ((!fmt->has_precision || length < (size_t)fmt->precision)
+		&& string[length])
 		length++;
 	return (length);
 }
@@ -33,9 +34,7 @@ int	ft_printf_print_string(t_printf *ctx, t_format *fmt, const char *string)
 
 	if (string == 0)
 		string = "(null)";
-	length = ft_local_strlen(string);
-	if (fmt->has_precision && fmt->precision < (int)length)
-		length = (size_t)fmt->precision;
+	length = ft_local_strlen(string, fmt);
 	padding = fmt->width - (int)length;
 	if (!(fmt->flags & FT_FLAG_LEFT)
 		&& ft_printf_putnchar(ctx, ' ', padding) < 0)


## `test(text): NUL 없는 제한 문자열 회귀 검증`

diff --git a/tests/test_ft_printf.c b/tests/test_ft_printf.c
index ae03124..b5965d8 100644
--- a/tests/test_ft_printf.c
+++ b/tests/test_ft_printf.c
@@ -187,6 +187,16 @@ static void	run_numeric_layout_cases(void)
 	EXPECT_PRINTF("hex-empty:'%#.0x'", 0u);
 }
 
+static void	run_text_differential_cases(void)
+{
+	char	bounded[3];
+
+	bounded[0] = 'a';
+	bounded[1] = 'b';
+	bounded[2] = 'c';
+	EXPECT_PRINTF("%.3s", bounded);
+}
+
 static void	run_error_cases(void)
 {
 	int	saved_stdout;
@@ -256,6 +266,7 @@ int	main(void)
 	run_bonus_cases();
 	run_parser_boundary_cases();
 	run_numeric_layout_cases();
+	run_text_differential_cases();
 	run_error_cases();
 	run_sigpipe_policy_case();
 	dprintf(STDERR_FILENO, "ft_printf tests passed\n");
