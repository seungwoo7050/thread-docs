# 측정–출력 2회 순회 포맷 파이프라인

## `feat(core): 리터럴과 퍼센트 출력 구현`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..d7edefd
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,4 @@
+*.a
+*.o
+*.d
+tests/bin/
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..90d8db3
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,30 @@
+NAME := libftprintf.a
+
+CC := cc
+CFLAGS := -Wall -Wextra -Werror
+CPPFLAGS := -Iinclude
+AR := ar
+ARFLAGS := rcs
+RM := rm -f
+
+SRC := src/ft_printf.c
+OBJ := $(SRC:.c=.o)
+
+all: $(NAME)
+
+$(NAME): $(OBJ)
+	$(AR) $(ARFLAGS) $@ $^
+
+%.o: %.c include/ft_printf.h
+	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
+
+clean:
+	$(RM) $(OBJ)
+	rm -rf tests/bin
+
+fclean: clean
+	$(RM) $(NAME)
+
+re: fclean all
+
+.PHONY: all clean fclean re
diff --git a/include/ft_printf.h b/include/ft_printf.h
new file mode 100644
index 0000000..7c0b3d5
--- /dev/null
+++ b/include/ft_printf.h
@@ -0,0 +1,6 @@
+#ifndef FT_PRINTF_H
+# define FT_PRINTF_H
+
+int	ft_printf(const char *format, ...);
+
+#endif
diff --git a/src/ft_printf.c b/src/ft_printf.c
new file mode 100644
index 0000000..9f4e8a3
--- /dev/null
+++ b/src/ft_printf.c
@@ -0,0 +1,51 @@
+#include "ft_printf.h"
+
+#include <limits.h>
+#include <stdarg.h>
+#include <unistd.h>
+
+static int	ft_write_count(const char *buffer, int length, int *count)
+{
+	int	written;
+
+	if (length <= 0)
+		return (0);
+	if (*count > INT_MAX - length)
+		return (-1);
+	written = (int)write(1, buffer, (size_t)length);
+	if (written < 0 || written != length)
+		return (-1);
+	*count += written;
+	return (0);
+}
+
+int	ft_printf(const char *format, ...)
+{
+	va_list	args;
+	int		count;
+
+	(void)args;
+	if (format == 0)
+		return (-1);
+	va_start(args, format);
+	count = 0;
+	while (*format)
+	{
+		if (*format == '%' && *(format + 1) == '%')
+		{
+			if (ft_write_count("%", 1, &count) < 0)
+				count = -1;
+			format += 2;
+		}
+		else
+		{
+			if (ft_write_count(format, 1, &count) < 0)
+				count = -1;
+			format++;
+		}
+		if (count < 0)
+			break ;
+	}
+	va_end(args);
+	return (count);
+}


## `feat(output): 출력 컨텍스트와 쓰기 API 추가`

diff --git a/Makefile b/Makefile
index 90d8db3..092971e 100644
--- a/Makefile
+++ b/Makefile
@@ -7,15 +7,17 @@ AR := ar
 ARFLAGS := rcs
 RM := rm -f
 
-SRC := src/ft_printf.c
+SRC := src/ft_printf.c \
+	src/ft_output.c
 OBJ := $(SRC:.c=.o)
+HEADER := include/ft_printf.h src/ft_printf_internal.h
 
 all: $(NAME)
 
 $(NAME): $(OBJ)
 	$(AR) $(ARFLAGS) $@ $^
 
-%.o: %.c include/ft_printf.h
+%.o: %.c $(HEADER)
 	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@
 
 clean:
diff --git a/src/ft_output.c b/src/ft_output.c
new file mode 100644
index 0000000..de1d83a
--- /dev/null
+++ b/src/ft_output.c
@@ -0,0 +1,42 @@
+#include "ft_printf_internal.h"
+
+#include <limits.h>
+#include <unistd.h>
+
+void	ft_printf_init(t_printf *ctx, int fd)
+{
+	ctx->fd = fd;
+	ctx->count = 0;
+	ctx->error = 0;
+}
+
+int	ft_printf_write(t_printf *ctx, const char *buffer, size_t length)
+{
+	ssize_t	written;
+
+	if (ctx->error)
+		return (-1);
+	while (length > 0)
+	{
+		written = write(ctx->fd, buffer, length);
+		if (written <= 0)
+		{
+			ctx->error = 1;
+			return (-1);
+		}
+		if (ctx->count > INT_MAX - (int)written)
+		{
+			ctx->error = 1;
+			return (-1);
+		}
+		ctx->count += (int)written;
+		buffer += written;
+		length -= (size_t)written;
+	}
+	return (0);
+}
+
+int	ft_printf_putchar(t_printf *ctx, char c)
+{
+	return (ft_printf_write(ctx, &c, 1));
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
new file mode 100644
index 0000000..524c99a
--- /dev/null
+++ b/src/ft_printf_internal.h
@@ -0,0 +1,19 @@
+#ifndef FT_PRINTF_INTERNAL_H
+# define FT_PRINTF_INTERNAL_H
+
+# include "ft_printf.h"
+
+# include <stddef.h>
+
+typedef struct s_printf
+{
+	int	fd;
+	int	count;
+	int	error;
+}	t_printf;
+
+void		ft_printf_init(t_printf *ctx, int fd);
+int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
+int			ft_printf_putchar(t_printf *ctx, char c);
+
+#endif


## `feat(parser): 포맷 필드 모델과 해석기 추가`

diff --git a/Makefile b/Makefile
index 092971e..210a17a 100644
--- a/Makefile
+++ b/Makefile
@@ -8,7 +8,8 @@ ARFLAGS := rcs
 RM := rm -f
 
 SRC := src/ft_printf.c \
-	src/ft_output.c
+	src/ft_output.c \
+	src/ft_parse.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_parse.c b/src/ft_parse.c
new file mode 100644
index 0000000..72f883d
--- /dev/null
+++ b/src/ft_parse.c
@@ -0,0 +1,74 @@
+#include "ft_printf_internal.h"
+
+#include <limits.h>
+
+static int	ft_is_digit(char c)
+{
+	return (c >= '0' && c <= '9');
+}
+
+static int	ft_flag_value(char c)
+{
+	if (c == '-')
+		return (FT_FLAG_LEFT);
+	if (c == '0')
+		return (FT_FLAG_ZERO);
+	if (c == '#')
+		return (FT_FLAG_HASH);
+	if (c == ' ')
+		return (FT_FLAG_SPACE);
+	if (c == '+')
+		return (FT_FLAG_PLUS);
+	return (0);
+}
+
+static int	ft_parse_decimal(const char **format, int *value)
+{
+	int	digit;
+
+	while (ft_is_digit(**format))
+	{
+		digit = **format - '0';
+		if (*value > (INT_MAX - digit) / 10)
+			return (-1);
+		*value = *value * 10 + digit;
+		(*format)++;
+	}
+	return (0);
+}
+
+void	ft_printf_init_format(t_format *fmt)
+{
+	fmt->flags = 0;
+	fmt->width = 0;
+	fmt->precision = 0;
+	fmt->has_precision = 0;
+	fmt->spec = '\0';
+}
+
+const char	*ft_printf_parse(const char *format, t_format *fmt)
+{
+	int	flag;
+
+	ft_printf_init_format(fmt);
+	flag = ft_flag_value(*format);
+	while (flag)
+	{
+		fmt->flags |= flag;
+		format++;
+		flag = ft_flag_value(*format);
+	}
+	if (ft_parse_decimal(&format, &fmt->width) < 0)
+		return (0);
+	if (*format == '.')
+	{
+		fmt->has_precision = 1;
+		format++;
+		if (ft_parse_decimal(&format, &fmt->precision) < 0)
+			return (0);
+	}
+	fmt->spec = *format;
+	if (*format)
+		format++;
+	return (format);
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 524c99a..e4c6e47 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -5,6 +5,12 @@
 
 # include <stddef.h>
 
+# define FT_FLAG_LEFT 1
+# define FT_FLAG_ZERO 2
+# define FT_FLAG_HASH 4
+# define FT_FLAG_SPACE 8
+# define FT_FLAG_PLUS 16
+
 typedef struct s_printf
 {
 	int	fd;
@@ -12,8 +18,19 @@ typedef struct s_printf
 	int	error;
 }	t_printf;
 
+typedef struct s_format
+{
+	int		flags;
+	int		width;
+	int		precision;
+	int		has_precision;
+	char	spec;
+}	t_format;
+
 void		ft_printf_init(t_printf *ctx, int fd);
 int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
+void		ft_printf_init_format(t_format *fmt);
+const char	*ft_printf_parse(const char *format, t_format *fmt);
 
 #endif


## `feat(core): 포맷 필드 해석을 출력 루프에 연결`

diff --git a/src/ft_printf.c b/src/ft_printf.c
index fd0e7bd..309fd28 100644
--- a/src/ft_printf.c
+++ b/src/ft_printf.c
@@ -6,6 +6,7 @@ int	ft_printf(const char *format, ...)
 {
 	va_list	args;
 	t_printf	ctx;
+	t_format	fmt;
 
 	(void)args;
 	if (format == 0)
@@ -20,6 +21,20 @@ int	ft_printf(const char *format, ...)
 				break ;
 			format += 2;
 		}
+		else if (*format == '%')
+		{
+			format = ft_printf_parse(format + 1, &fmt);
+			if (format == 0)
+			{
+				ctx.error = 1;
+				break ;
+			}
+			if (ft_printf_putchar(&ctx, '%') < 0)
+				break ;
+			if (fmt.spec != '\0' && fmt.spec != '%'
+				&& ft_printf_putchar(&ctx, fmt.spec) < 0)
+				break ;
+		}
 		else
 		{
 			if (ft_printf_putchar(&ctx, *format) < 0)


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


## `feat(decimal): 부호 있는·없는 10진수 출력 추가`

diff --git a/Makefile b/Makefile
index fb35e4e..bdd8f04 100644
--- a/Makefile
+++ b/Makefile
@@ -11,7 +11,8 @@ SRC := src/ft_printf.c \
 	src/ft_output.c \
 	src/ft_parse.c \
 	src/ft_dispatch.c \
-	src/ft_text.c
+	src/ft_text.c \
+	src/ft_number.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_dispatch.c b/src/ft_dispatch.c
index dede10c..9bb1e13 100644
--- a/src/ft_dispatch.c
+++ b/src/ft_dispatch.c
@@ -6,6 +6,11 @@ int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
 		return (ft_printf_print_char(ctx, fmt, va_arg(*args, int)));
 	if (fmt->spec == 's')
 		return (ft_printf_print_string(ctx, fmt, va_arg(*args, char *)));
+	if (fmt->spec == 'd' || fmt->spec == 'i')
+		return (ft_printf_print_signed(ctx, fmt, va_arg(*args, int)));
+	if (fmt->spec == 'u')
+		return (ft_printf_print_unsigned(ctx, fmt,
+				va_arg(*args, unsigned int)));
 	if (fmt->spec == '%')
 		return (ft_printf_print_percent(ctx, fmt));
 	if (ft_printf_putchar(ctx, '%') < 0)
diff --git a/src/ft_number.c b/src/ft_number.c
new file mode 100644
index 0000000..2290926
--- /dev/null
+++ b/src/ft_number.c
@@ -0,0 +1,44 @@
+#include "ft_printf_internal.h"
+
+static int	ft_print_unsigned_digits(t_printf *ctx, unsigned long number)
+{
+	char	buffer[20];
+	int		index;
+
+	index = 0;
+	if (number == 0)
+		buffer[index++] = '0';
+	while (number > 0)
+	{
+		buffer[index++] = (char)('0' + number % 10);
+		number /= 10;
+	}
+	while (index > 0)
+	{
+		index--;
+		if (ft_printf_putchar(ctx, buffer[index]) < 0)
+			return (-1);
+	}
+	return (0);
+}
+
+int	ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number)
+{
+	long	value;
+
+	(void)fmt;
+	value = (long)number;
+	if (value < 0)
+	{
+		if (ft_printf_putchar(ctx, '-') < 0)
+			return (-1);
+		value = -value;
+	}
+	return (ft_print_unsigned_digits(ctx, (unsigned long)value));
+}
+
+int	ft_printf_print_unsigned(t_printf *ctx, t_format *fmt, unsigned int number)
+{
+	(void)fmt;
+	return (ft_print_unsigned_digits(ctx, (unsigned long)number));
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 70695ad..c2da9dc 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -38,5 +38,8 @@ int			ft_printf_print_char(t_printf *ctx, t_format *fmt, int c);
 int			ft_printf_print_string(t_printf *ctx, t_format *fmt,
 				const char *string);
 int			ft_printf_print_percent(t_printf *ctx, t_format *fmt);
+int			ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number);
+int			ft_printf_print_unsigned(t_printf *ctx, t_format *fmt,
+				unsigned int number);
 
 #endif


## `feat(hex): 16진수와 포인터 출력 추가`

diff --git a/Makefile b/Makefile
index bdd8f04..fbaddff 100644
--- a/Makefile
+++ b/Makefile
@@ -12,7 +12,8 @@ SRC := src/ft_printf.c \
 	src/ft_parse.c \
 	src/ft_dispatch.c \
 	src/ft_text.c \
-	src/ft_number.c
+	src/ft_number.c \
+	src/ft_hex.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_dispatch.c b/src/ft_dispatch.c
index 9bb1e13..b7b9a7a 100644
--- a/src/ft_dispatch.c
+++ b/src/ft_dispatch.c
@@ -11,6 +11,10 @@ int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
 	if (fmt->spec == 'u')
 		return (ft_printf_print_unsigned(ctx, fmt,
 				va_arg(*args, unsigned int)));
+	if (fmt->spec == 'x' || fmt->spec == 'X')
+		return (ft_printf_print_hex(ctx, fmt, va_arg(*args, unsigned int)));
+	if (fmt->spec == 'p')
+		return (ft_printf_print_pointer(ctx, fmt, va_arg(*args, void *)));
 	if (fmt->spec == '%')
 		return (ft_printf_print_percent(ctx, fmt));
 	if (ft_printf_putchar(ctx, '%') < 0)
diff --git a/src/ft_hex.c b/src/ft_hex.c
new file mode 100644
index 0000000..132d785
--- /dev/null
+++ b/src/ft_hex.c
@@ -0,0 +1,41 @@
+#include "ft_printf_internal.h"
+
+#include <stdint.h>
+
+static int	ft_print_base(t_printf *ctx, unsigned long number, const char *digits)
+{
+	char	buffer[2 + sizeof(unsigned long) * 2];
+	int		index;
+
+	index = 0;
+	if (number == 0)
+		buffer[index++] = '0';
+	while (number > 0)
+	{
+		buffer[index++] = digits[number % 16];
+		number /= 16;
+	}
+	while (index > 0)
+	{
+		index--;
+		if (ft_printf_putchar(ctx, buffer[index]) < 0)
+			return (-1);
+	}
+	return (0);
+}
+
+int	ft_printf_print_hex(t_printf *ctx, t_format *fmt, unsigned int number)
+{
+	if (fmt->spec == 'X')
+		return (ft_print_base(ctx, (unsigned long)number, "0123456789ABCDEF"));
+	return (ft_print_base(ctx, (unsigned long)number, "0123456789abcdef"));
+}
+
+int	ft_printf_print_pointer(t_printf *ctx, t_format *fmt, void *pointer)
+{
+	(void)fmt;
+	if (ft_printf_write(ctx, "0x", 2) < 0)
+		return (-1);
+	return (ft_print_base(ctx, (unsigned long)(uintptr_t)pointer,
+			"0123456789abcdef"));
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index c2da9dc..4bc3bb9 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -41,5 +41,9 @@ int			ft_printf_print_percent(t_printf *ctx, t_format *fmt);
 int			ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number);
 int			ft_printf_print_unsigned(t_printf *ctx, t_format *fmt,
 				unsigned int number);
+int			ft_printf_print_hex(t_printf *ctx, t_format *fmt,
+				unsigned int number);
+int			ft_printf_print_pointer(t_printf *ctx, t_format *fmt,
+				void *pointer);
 
 #endif


## `fix(format): 지원 문법과 전체 출력 크기 선검증`

diff --git a/Makefile b/Makefile
index 730b260..552c46d 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,7 @@ RM := rm -f
 SRC := src/ft_printf.c \
 	src/ft_output.c \
 	src/ft_parse.c \
+	src/ft_measure.c \
 	src/ft_dispatch.c \
 	src/ft_text.c \
 	src/ft_numeric_layout.c \
diff --git a/src/ft_measure.c b/src/ft_measure.c
new file mode 100644
index 0000000..77e844d
--- /dev/null
+++ b/src/ft_measure.c
@@ -0,0 +1,188 @@
+#include "ft_printf_internal.h"
+
+#include <limits.h>
+#include <stdint.h>
+
+static int	ft_is_supported_specifier(char spec)
+{
+	return (spec == 'c' || spec == 's' || spec == 'd' || spec == 'i'
+		|| spec == 'u' || spec == 'x' || spec == 'X' || spec == 'p'
+		|| spec == '%');
+}
+
+static int	ft_add_length(size_t *total, size_t amount)
+{
+	if (amount > (size_t)INT_MAX || *total > (size_t)INT_MAX - amount)
+		return (-1);
+	*total += amount;
+	return (0);
+}
+
+static int	ft_decimal_length(unsigned long number)
+{
+	int	length;
+
+	length = 1;
+	while (number >= 10)
+	{
+		number /= 10;
+		length++;
+	}
+	return (length);
+}
+
+static int	ft_hex_length(unsigned long number)
+{
+	int	length;
+
+	length = 1;
+	while (number >= 16)
+	{
+		number /= 16;
+		length++;
+	}
+	return (length);
+}
+
+static int	ft_measure_numeric(t_format *fmt, int prefix_length,
+		int digit_length, int is_zero, size_t *length)
+{
+	size_t	content_length;
+
+	if (fmt->has_precision && fmt->precision == 0 && is_zero)
+		digit_length = 0;
+	content_length = (size_t)digit_length;
+	if (fmt->has_precision && fmt->precision > digit_length)
+		content_length = (size_t)fmt->precision;
+	if (ft_add_length(&content_length, (size_t)prefix_length) < 0)
+		return (-1);
+	if ((size_t)fmt->width > content_length)
+		content_length = (size_t)fmt->width;
+	*length = content_length;
+	return (0);
+}
+
+static int	ft_measure_string(t_format *fmt, const char *string,
+		size_t *length)
+{
+	size_t	string_length;
+
+	if (string == 0)
+		string = "(null)";
+	string_length = 0;
+	while ((!fmt->has_precision
+			|| string_length < (size_t)fmt->precision)
+		&& string[string_length])
+	{
+		if (string_length == (size_t)INT_MAX)
+			return (-1);
+		string_length++;
+	}
+	if ((size_t)fmt->width > string_length)
+		string_length = (size_t)fmt->width;
+	*length = string_length;
+	return (0);
+}
+
+static int	ft_measure_signed(t_format *fmt, int number, size_t *length)
+{
+	long			value;
+	unsigned long	magnitude;
+	int				prefix_length;
+
+	value = (long)number;
+	prefix_length = 0;
+	if (value < 0)
+	{
+		prefix_length = 1;
+		magnitude = (unsigned long)(-(value + 1)) + 1;
+	}
+	else
+	{
+		magnitude = (unsigned long)value;
+		if (fmt->flags & (FT_FLAG_PLUS | FT_FLAG_SPACE))
+			prefix_length = 1;
+	}
+	return (ft_measure_numeric(fmt, prefix_length,
+			ft_decimal_length(magnitude), magnitude == 0, length));
+}
+
+static int	ft_measure_hex(t_format *fmt, unsigned long number,
+		int is_pointer, size_t *length)
+{
+	int	prefix_length;
+
+	prefix_length = 0;
+	if (is_pointer || ((fmt->flags & FT_FLAG_HASH) && number != 0))
+		prefix_length = 2;
+	return (ft_measure_numeric(fmt, prefix_length, ft_hex_length(number),
+			number == 0, length));
+}
+
+static int	ft_measure_conversion(t_format *fmt, va_list *args,
+		size_t *length)
+{
+	unsigned int	unsigned_number;
+
+	if (fmt->spec == 'c')
+	{
+		(void)va_arg(*args, int);
+		*length = 1;
+	}
+	else if (fmt->spec == 's')
+		return (ft_measure_string(fmt, va_arg(*args, char *), length));
+	else if (fmt->spec == 'd' || fmt->spec == 'i')
+		return (ft_measure_signed(fmt, va_arg(*args, int), length));
+	else if (fmt->spec == 'u')
+	{
+		unsigned_number = va_arg(*args, unsigned int);
+		return (ft_measure_numeric(fmt, 0,
+				ft_decimal_length((unsigned long)unsigned_number),
+				unsigned_number == 0, length));
+	}
+	else if (fmt->spec == 'x' || fmt->spec == 'X')
+		return (ft_measure_hex(fmt,
+				(unsigned long)va_arg(*args, unsigned int), 0, length));
+	else if (fmt->spec == 'p')
+		return (ft_measure_hex(fmt,
+				(unsigned long)(uintptr_t)va_arg(*args, void *), 1, length));
+	else
+		*length = 1;
+	if ((size_t)fmt->width > *length)
+		*length = (size_t)fmt->width;
+	return (0);
+}
+
+int	ft_printf_measure(const char *format, va_list *args)
+{
+	t_format	fmt;
+	size_t		conversion_length;
+	size_t		total;
+
+	total = 0;
+	while (*format)
+	{
+		if (*format == '%' && *(format + 1) == '%')
+		{
+			if (ft_add_length(&total, 1) < 0)
+				return (-1);
+			format += 2;
+		}
+		else if (*format == '%')
+		{
+			format = ft_printf_parse(format + 1, &fmt);
+			if (format == 0 || !ft_is_supported_specifier(fmt.spec)
+				|| ft_measure_conversion(&fmt, args,
+					&conversion_length) < 0
+				|| ft_add_length(&total, conversion_length) < 0)
+				return (-1);
+		}
+		else
+		{
+			if (ft_add_length(&total, 1) < 0)
+				return (-1);
+			format++;
+		}
+	}
+	return ((int)total);
+}
diff --git a/src/ft_printf.c b/src/ft_printf.c
index be2dedd..159a82b 100644
--- a/src/ft_printf.c
+++ b/src/ft_printf.c
@@ -3,13 +3,21 @@
 int	ft_printf(const char *format, ...)
 {
 	va_list	args;
+	va_list	measure_args;
 	t_printf	ctx;
 	t_format	fmt;
 
-	(void)args;
 	if (format == 0)
 		return (-1);
 	va_start(args, format);
+	va_copy(measure_args, args);
+	if (ft_printf_measure(format, &measure_args) < 0)
+	{
+		va_end(measure_args);
+		va_end(args);
+		return (-1);
+	}
+	va_end(measure_args);
 	ft_printf_init(&ctx, 1);
 	while (*format)
 	{
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 5bb0aee..088ffcd 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -32,6 +32,7 @@ void		ft_printf_init(t_printf *ctx, int fd);
 int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
 int			ft_printf_putnchar(t_printf *ctx, char c, int length);
+int			ft_printf_measure(const char *format, va_list *args);
 int			ft_printf_write_numeric_layout(t_printf *ctx, t_format *fmt,
 				const char *prefix, const char *digits, int digit_len,
 				int is_zero);
